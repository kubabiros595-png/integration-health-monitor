# Jak to działa — prostym językiem

Ten plik jest dla mnie. Wszystkie dane w projekcie są fikcyjne.

## O co chodzi w projekcie

Firma SaaS dostaje od sklepów internetowych klientów **webhooki**: krótkie wiadomości wysyłane automatycznie, gdy coś się stanie (ktoś złożył zamówienie, klient zmienił zgodę na maile). Problem: gdy integracja się psuje, często psuje się **po cichu**. Nic nie wybucha, po prostu przestają przychodzić dane, a klient dowiaduje się o tym po tygodniu, z wyników.

System robi trzy rzeczy:
1. Bezpiecznie przyjmuje webhooki (sprawdza, czy są prawdziwe, świeże, poprawne i nie powtórzone).
2. Co godzinę sprawdza, czy któraś integracja nie ucichła albo nie zaczęła sypać błędami.
3. Gdy coś wykryje, przygotowuje dla CSM-a szkic maila do klienta. Szkic czeka na akceptację, nic nie idzie samo.

## Czym jest HMAC

Wyobraź sobie, że sklep i my znamy wspólne hasło (**sekret**). Nikt inny go nie zna.

Sklep, wysyłając wiadomość, liczy z niej i z sekretu „odcisk palca”: ciąg 64 znaków. To jest **podpis HMAC-SHA256**. Wysyła go w nagłówku razem z wiadomością.

My robimy to samo: bierzemy wiadomość, którą dostaliśmy, i nasz sekret, liczymy odcisk. Jeśli wyszedł identyczny, wiemy dwie rzeczy:
- wiadomość wysłał ktoś, kto zna sekret (czyli sklep),
- nikt jej po drodze nie zmienił, bo zmiana nawet jednej litery daje zupełnie inny odcisk.

Sekret nigdy nie jest wysyłany. Wysyłany jest tylko odcisk.

**Dlaczego „surowe body” (raw body).** Podpis liczymy z dokładnie tych bajtów, które przyszły. Gdyby n8n najpierw rozpakował JSON i spakował go z powrotem, mogłyby się zmienić spacje albo kolejność pól, i podpis by się nie zgodził, mimo że wiadomość jest prawdziwa. Test S07 to pokazuje: ta sama treść, ale ładnie sformatowana, daje 401.

**Gdzie jest sekret.** W zmiennych n8n (`IHM_SECRET_...`), nie w kodzie i nie w repo. W eksportach workflowów widać tylko nazwę zmiennej.

## Po co znacznik czasu

Ktoś mógłby przechwycić prawdziwą, poprawnie podpisaną wiadomość i wysłać ją jeszcze raz za godzinę (**atak powtórzeniowy**, replay). Podpis by się zgadzał, bo wiadomość jest prawdziwa.

Dlatego sklep dokleja do podpisu **znacznik czasu** (sekundy od 1970 roku). Podpis obejmuje czas i treść razem, więc czasu nie da się podmienić (test S08). My odrzucamy wszystko starsze niż 5 minut (testy T01, T02) i wszystko, co jest więcej niż 5 minut „z przyszłości” (T03). Margines w drugą stronę jest po to, że zegary serwerów nigdy nie są idealnie zgrane (V10).

## Czym jest idempotency

Sklepy wysyłają ten sam webhook ponownie, gdy nie są pewne, czy dotarł (np. nasza odpowiedź się spóźniła). To normalne i dobre. Ale gdybyśmy każde powtórzenie zapisywali, jedno zamówienie liczyłoby się dwa razy.

**Idempotency** znaczy: zrobienie czegoś dwa razy daje ten sam efekt co raz. U nas: każde zdarzenie ma `event_id`. Zanim zapiszemy, sprawdzamy, czy ten sklep już przysłał to `event_id`. Jeśli tak, odpowiadamy 200 (żeby sklep przestał ponawiać), wpisujemy do logu „duplicate” i nic więcej nie robimy.

Klucz to **sklep + event_id**, bo dwa różne sklepy mogą przypadkiem użyć tego samego numeru (test D05).

**Wyścig (dwa identyczne żądania w tej samej chwili).** Oba sprawdzają tabelę, zanim którekolwiek zdążyło się zapisać, więc oba widzą „nowe”. Pierwsze testy to pokazały. Dlatego po zapisie jest drugie sprawdzenie: czytam wszystkie wiersze z tym samym sklepem i `event_id`. Wygrywa wiersz z najniższym numerem, a późniejsza kopia sama się usuwa i odpowiada „duplicate”. Test D06 wysyła to samo żądanie 3 razy naraz: 1 przyjęte, 2 duplikaty, 1 zapis.

Duplikaty **nie są błędami**. Detektor ich nie liczy do odsetka błędów (scenariusz M11).

## Dlaczego odpowiedź 200 musi być szybka

Sklep czeka na naszą odpowiedź tylko kilka sekund. Jeśli się spóźnimy, uzna, że wiadomość nie doszła, i wyśle ją ponownie, a po kilku porażkach może w ogóle wyłączyć webhook. Wtedy mamy właśnie cichą awarię.

Najpierw sprawdzamy rozmiar: treść powyżej 64 KB dostaje od razu **413** (za duże), zanim liczymy jakikolwiek podpis. Prawdziwe zdarzenie ma kilkaset bajtów, więc to tania ochrona przed zasypaniem dużymi żądaniami.

Odbiór robi tylko to, co konieczne: podpis, walidacja, sprawdzenie duplikatu, zapis. Odpowiedź idzie **zaraz po zapisie**, a log piszemy już po odpowiedzi. 200 zawsze znaczy „mamy to zapisane”. Cięższe rzeczy (analiza, AI, raporty) dzieją się osobno, w innych workflowach.

## Jakie odpowiedzi dostaje sklep

| Sytuacja | Kod | Co dostaje sklep |
| --- | --- | --- |
| Zły podpis, nieznany sklep, zły lub stary czas | 401 | tylko „unauthorized”, bez szczegółów (nie podpowiadamy atakującemu) |
| Brak pola, zły typ, zły JSON | 400 | lista problemów, żeby programista sklepu wiedział, co poprawić |
| Treść powyżej 64 KB | 413 | „payload too large” |
| Duplikat | 200 | „duplicate” |
| Wszystko dobrze | 200 | „accepted” |

Każde żądanie, także odrzucone, trafia do logu z powodem.

## Jak działa detektor ciszy

Prosta reguła „alarm po godzinie ciszy” nie działa, bo sklepy mają różny rytm. Duży sklep wysyła zdarzenie co 5 minut, mały co 2 godziny. Godzina ciszy u małego to normalka (M04), a u dużego to awaria.

Dlatego dla każdej integracji osobno:
1. Biorę zdarzenia z ostatnich 7 dni i liczę przerwy między nimi.
2. Biorę **90. percentyl** przerw: przerwę, od której 90% przerw jest krótszych. To jest „zwykła przerwa” tego sklepu.
3. Granica ciszy = 3 × zwykła przerwa, ale nigdy mniej niż 60 minut.
4. Jeśli od ostatniego zdarzenia minęło więcej niż granica, jest alert.

Przykład: zwykła przerwa 5 min → granica 60 min (bo 3 × 5 = 15, ale minimum to 60). Cisza 59 min nie daje alertu (M06), 61 min daje (M07).

Wyjątki:
- Mniej niż 20 zdarzeń w historii: za mało danych, nie zgadujemy (M12).
- Zero zdarzeń przez 7 dni: zawsze alert (M13). To jest dokładnie przypadek „0 odbiorców i nikt nie zauważył”.

## Jak działa alert o skoku błędów

W ostatniej godzinie liczę: odrzucone / (przyjęte + odrzucone). Alert, gdy wynik jest **powyżej 10%** i było co najmniej 10 żądań. Przy 3 żądaniach dwa błędy to 66%, ale to za mało, żeby coś stwierdzić (M10).

## Co robi AI, a czego nie robi

- **Reguły w kodzie** decydują, czy jest problem. AI nie decyduje o niczym.
- Kod wybiera **fakty** (liczby) i **co klient ma sprawdzić** (np. przy błędach podpisu: czy zmienił sekret).
- **Claude** tylko ubiera to w krótki mail po angielsku: co się stało, co sprawdzić, następny krok.
- Potem **kod sprawdza tekst**: każda liczba musi pochodzić z faktów, musi być główna liczba, nie może być sekretu ani niczego, co wygląda na klucz lub podpis, nie może być nazwy innego klienta. Blokuje też błędy CSM-a: obietnicę naprawy albo terminu, propozycję calla i darmową pracę.
- Alert zapisuje się jako „pending review”. Do klienta nic nie idzie samo.
- Ten sam problem nie daje nowego alertu przez 6 godzin, więc drugie uruchomienie nie wywołuje AI ani razu.

## Słowniczek

- **Webhook**: automatyczna wiadomość „coś się stało”, wysyłana z jednego systemu do drugiego.
- **Endpoint**: adres, pod który przychodzą webhooki.
- **Payload / body**: treść wiadomości (tutaj JSON).
- **Nagłówek (header)**: dodatkowe informacje obok treści, np. podpis i czas.
- **401**: „nie wiem, kim jesteś”. **400**: „twoja wiadomość jest źle zbudowana”. **200**: „OK”.

## 5 pytań rekrutacyjnych (po angielsku)

**1. Why do you verify the signature on the raw body and not on the parsed JSON?**
Because the signature is calculated on the exact bytes the sender sent. Parsing and re-serialising JSON can change spacing or key order, so a genuine request would fail. One of my tests sends the same content pretty-printed, and it is correctly rejected with 401.

**2. A valid signature proves the request is genuine. Why do you also need a timestamp?**
Without it, someone who captured a real request could replay it later and the signature would still match. The timestamp is part of the signed string, so it cannot be changed, and anything older than five minutes is rejected. Exact repeats inside those five minutes are caught by the event_id check.

**3. What is idempotency and how did you implement it?**
Processing the same event twice has the same effect as processing it once. Senders retry when they are unsure, so duplicates are normal. Before saving, I look up shop plus event_id. If it exists, I return 200 so they stop retrying, log it as a duplicate and do nothing else. My first tests showed that two identical requests at the same moment could both pass that lookup, so I added a check after saving: the first stored copy wins and any later copy deletes itself. A test sends three copies at once and only one is stored. In production I would use a unique constraint in the database instead.

**4. How does the silence detector avoid false alarms?**
It learns each integration's own rhythm: the 90th percentile of the gaps between events over seven days. It alerts only after three times that gap, with a 60-minute floor. A shop that sends every two hours is not flagged after one quiet hour, and a new integration with fewer than 20 events is not judged at all. In my tests that gave zero false alarms and zero missed alerts.

**5. Why use AI only for the message and not for the detection?**
Detection has to be predictable and explainable, so it is plain rules. The AI writes the wording, and code then checks that every number comes from the computed facts and that no secret, key or other client's name is in the text. A person approves the draft before anything reaches the client.
