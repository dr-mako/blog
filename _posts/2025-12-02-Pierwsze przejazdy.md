---
layout: post
title: "Pierwsze przejazdy!"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Surowe wyniki i wyznaczanie trajektorii w postprocesingu <!--more-->

### 1) Surowe dane pomiarowe
Zmienne sterujące i obserwowane zapisuję do pliku „out.txt” w formacie NDJSON (Newline‑Delimited JSON). Każda linia to niezależny obiekt JSON, co umożliwia strumieniowe przetwarzanie bez ładowania całego pliku do pamięci. Dzięki temu łatwo scalać różne strumienie logów oraz wykonywać filtrację i synchronizację w czasie. Dane z napędów i serw rejestruję co 10 ms.

Poniżej fragment surowego logu:

{"type": "MOTOR", "time": "44.535256928", "data": {"T": 20010, "id": 1, "typ": 400, "spd": 222, "crt": 351, "act": 1, "tep": 19, "err": 0}}
{"type": "SERVO", "time": "44.535667104", "data": {"setAngle": "14.557342529296875", "servo_1": "12.74", "servo_2": "-12.83", "feedback_type": "FBK"}}
{"type": "MOTOR", "time": "44.544571104", "data": {"T": 20010, "id": 4, "typ": 400, "spd": 215, "crt": 249, "act": 1, "tep": 18, "err": 0}}
{"type": "MOTOR", "time": "44.556052736", "data": {"T": 20010, "id": 3, "typ": 400, "spd": -290, "crt": -651, "act": 1, "tep": 19, "err": 0}}
{"type": "MOTOR", "time": "44.56589152", "data": {"T": 20010, "id": 2, "typ": 400, "spd": -291, "crt": -717, "act": 1, "tep": 19, "err": 0}}
{"type": "SERVO", "time": "44.574356032", "data": {"setAngle": "14.557342529296875", "servo_1": "13.45", "servo_2": "-13.54", "feedback_type": "FBK"}}
{"type": "MOTOR", "time": "44.57628272", "data": {"T": 20010, "id": 1, "typ": 400, "spd": 213, "crt": 266, "act": 1, "tep": 19, "err": 0}}
{"type": "MOTOR", "time": "44.586096576", "data": {"T": 20010, "id": 4, "typ": 400, "spd": 217, "crt": 235, "act": 1, "tep": 18, "err": 0}}
{"type": "MOTOR", "time": "44.596907488", "data": {"T": 20010, "id": 3, "typ": 400, "spd": -274, "crt": -522, "act": 1, "tep": 19, "err": 0}}
{"type": "MOTOR", "time": "44.606897056", "data": {"T": 20010, "id": 2, "typ": 400, "spd": -271, "crt": -708, "act": 1, "tep": 19, "err": 0}}
{"type": "SERVO", "time": "44.614625568", "data": {"setAngle": "14.557342529296875", "servo_1": "14.15", "servo_2": "-14.15", "feedback_type": "FBK"}}

#### 2) Struktura i znaczenie pól
Każdy rekord zawiera typ, znacznik czasu i sekcję danych. Pole type określa źródło pomiaru: „MOTOR” dla napędów kół (DDSM400) oraz „SERVO” dla układu skrętu (ST3215). Pole time to znacznik czasu w sekundach zapisany jako tekst; w analizie rzutuję go na liczbę zmiennoprzecinkową i używam do synchronizacji. Pole data zawiera właściwe wartości telemetryczne, których zestaw zależy od typu rekordu.

MOTOR: identyfikator napędu (id), tryb/typ ramki (typ), prędkość obrotowa (spd), prąd (crt), temperatura (tep), flaga aktywności (act), kod błędu (err).

SERVO: zadany kąt (setAngle) oraz dwa kanały sprzężenia zwrotnego z enkoderów (servo_1, servo_2), plus typ ramki (feedback_type).

W feedback_type rozróżniam:
- ACK — potwierdzenie przyjęcia komendy (po ustawieniu pozycji/prędkości/konfiguracji),

- FBK — ramka telemetryczna ze sprzężenia zwrotnego (bieżący stan serwa).

To właśnie FBK wykorzystuję w postprocessingu do wyznaczania kąta skrętu δ (konwersja ze stopni na radiany i ewentualna fuzja kanałów servo_1/servo_2).

#### 3) Parsowanie i synchronizacja
Plik przetwarzam strumieniowo. Każdą linię parsuję jako JSON i — wg pola type — kieruję do dekodera MOTOR lub SERVO. Następnie:

- normalizuję jednostki (kąty: deg→rad; prędkości: na SI),

- łączę strumienie po czasie, otrzymując spójną serię do obliczeń trajektorii,

- w razie potrzeby filtruję anomalie (np. rozbieżności servo_1 vs servo_2), wygładzam szum i wykonuję interpolację,

- resampluję do interwału 20 ms (50 Hz), aby ujednolicić krok obliczeń.

#### 4) Jednostki i wagi według dokumentacji. Obróbka danych
Skale wynikają z dokumentacji:

- DDSM400: prędkość spd w dziesiątych części rpm (0.1 rpm), prąd crt w mA.

- ST3215: kąty w stopniach.

Konwersje do SI:

- prędkość liniowa koła: \(v = 2\pi R_w \cdot \mathrm{spd}/60\)  (po uwzględnieniu skali 0.1 rpm),

- kąt skrętu: \(\delta = \mathrm{deg}\cdot \pi/180\).

Model obliczeniowy wymaga jednej pary zmiennych sterujących {prędkość, kąt skrętu}. W logach mam cztery prędkości silników i dwa kąty skrętu. W tej fazie przyjmuję uśrednianie:

- prędkość pojazdu jako średnia z prędkości kół (ze znakiem i jednostkami po konwersji),

- kąt skrętu jako średnia z kanałów FBK (servo_1, servo_2) po konwersji do rad.

Tak przygotowany sygnał zasila model 4WS do wyznaczania trajektorii środka geometrycznego w globalnym układzie.

#### 5) Przejazd 1
Pierwszy rysunek pokazuje wykresy surowych serii: prędkości i prądy silników oraz kąty skrętu serw — bez ingerencji, w jednostkach z plików.

<img src="{{ 'assets/images/Przejazd/Przejazd0bezobrobki.png' | relative_url }}" alt="Przejazd0bezobrobki" style="width:100%; max-width:100%; height:auto;" />

Drugi rysunek przedstawia uśrednione wartości prędkości pojazdu i kątów skrętu wyrażone w jednostkach SI.

<img src="{{ 'assets/images/Przejazd/Jazda0SI.png' | relative_url }}" alt="Jazda0SI" style="width:100%; max-width:100%; height:auto;" />

Trzeci rysunek to uzyskana trajektoria w globalnym układzie współrzędnych. W chwili początkowej pojazd znajdował się w punkcie [0, 0]. Po wykonaniu pętli wrócił w przybliżeniu w to samo miejsce.

<img src="{{ 'assets/images/Przejazd/Trajekt0.png' | relative_url }}" alt="Trajekt0" style="width:100%; max-width:100%; height:auto;" />

#### 6) Przejazd 2
Drugi przejazd był dłuższy — pojazd wykonał dodatkową pętlę w drugim pomieszczeniu. Jak poprzednio, wszystkie sygnały rejestrowałem. Poniżej surowe serie w jednostkach z plików. 

<img src="{{ 'assets/images/Przejazd/Przejazd1bezobrobki.png' | relative_url }}" alt="Przejazd1bezobrobki" style="width:100%; max-width:100%; height:auto;" />

Uśrednione wartości w jednostkach SI:

<img src="{{ 'assets/images/Przejazd/Jazda1SI.png' | relative_url }}" alt="Jazda1SI" style="width:100%; max-width:100%; height:auto;" />

Trajektoria w układzie globalnym. Start z [0, 0]; po pętli powrót w pobliże punktu startu.

<img src="{{ 'assets/images/Przejazd/Trajekt1.png' | relative_url }}" alt="Trajekt1" style="width:100%; max-width:100%; height:auto;" />

#### 7) Wnioski
Podczas jazd pojazd poruszał się z minimalną prędkością około 0.15 m/s. Kąty skrętu były umiarkowane (±15°). W tych warunkach poślizgi boczne w łukach były niewielkie. Dzięki brakowi luzów w układzie kierowniczym pojazd poruszał się po płaszczyźnie drogi z wysoką powtarzalnością. Enkodery o rozdzielczości 4096 imp/obr zapewniły dokładne odczyty prędkości i kątów. Testy potwierdziły, że zakładane efekty modelowania i postprocessingu zostały osiągnięte.

Przeprowadziłem również dłuższe jazdy. W sesjach do 30 minut. W tych przejazdach, mikrokomputer Jetson ani razu się nie zawiesił podczas rejestracji danych, co potwierdza poprawność projektu pod względem: komunikacji i zasilania.

#### 8) Co dalej
W kolejnym wpisie pokażę rachunek niepewności i propagację błędu dla zastosowanego modelu, a następnie włączę obserwator (filtrację) do bieżącej estymacji trajektorii.
