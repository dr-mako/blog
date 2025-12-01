---
layout: post
title: "Etapy budowy platformy modelu"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Pokażę, etapy budowy platformy modelu — listę elementów, oraz jakie wydruki 3D przygotować, żeby złożyć prototyp <!--more-->

Tablica ukazuje najważniejsze elementy

| Element               | Ilość  | Uwagi                                                 |
| --------------------- | ------ | ----------------------------------------------------- |
| DDSM400 Motors        | 4 szt. | Do napędu bezpośredniego 4 WD (silniki w kołach)      |
| ESP32 Hub Motors      | 1 szt. | Kontroler sterowania dla 4x DDSM400                   |
| ST3215 Servo          | 2 szt. | Po jednym serwomechanizmie dla osi przedniej i tylnej |
| ESP32 Bus Servo       | 1 szt. | Kontroler sterowania dla 2x ST3215 Servo              |
| Kamera                | 2 szt. | Jedna przód + druga tył                               |
| Jetson Orin Nano      | 1 szt. | Host + obliczenia AI (wideo, czujniki)                |
| Lidary (opcjonalnie)  | 4 szt. | Do wykrywania odległości (rozbudowa później)          |
| Bateria LiPo 6S       | 1 szt  | Zasilanie główne 24V, 11 Ah, 110C                     |
| Przetwornica Buck 12V | 2 szt  | Victron Orion 36–18V/12V 20A 240W                     |
| Ładowarka dla LiPo    | 1 szt  | Wymagany balancer                                     |


### 1) Zawieszenie i koła
Do napędu używam czterech silników DDSM400, czyli napędów bezpośrednich z wbudowanym enkoderem. Do nich dokładamy firmowe zawieszenie UGV Suspension (B). To gotowy, sprężynowany moduł, który pozwala kołu pracować na nierównościach. Do kompletu mam czarny, drukowany element – to customowa „główka” zawieszenia (element drukowany na drukarce 3D), która łączy moduł z metalową osią obrotową. Oś jest stalowa, przechodzi przez dwa łożyska i daje płynny obrót bez luzów. Mechanicznie można by obracać koło do pełnego obrotu, ale w praktyce ograniczam ten kąt programowo do 35 stopni. Ta waqrtość zapewnia wystarczającą skrętność samochodu oraz gwarantuje, że przewody zasilania mają zapas i nie skręcają się na siłę. 
Na zdjęciach widać cztery gotowe wózki kół. Każdy ma tę samą budowę: na dole koło z silnikiem DDSM400, nad nim czerwone sprężyny firmowego zawieszenia, a na górze moja "customowa" czarna główka z gniazdem pod stalową oś. Przewód z silnika wyprowadzamy górą przez specjalnie przygotowany otwór, żeby potem łatwo prowadzić go do elektroniki. Dzięki takiej powtarzalności wszystkie narożniki montuje się tak samo, co przyspiesza składanie.
Co jest tu ważne: osie są mocowane do główki zawieszenia za pomocą sztywnego sprzęgła kołnierzowego, które jest blokowane wkrętami. Połączenie musi być bezluzowe tzn. osie muszą być odpowiednio ścięte w miejscu mocowania kołnierza. Ilustracje:

<div style="display:flex; gap:12px; flex-wrap:wrap;">
  <img src="{{ 'assets/images/etapy_bud/zaw1.jpg' | relative_url }}" alt="zaw1" style="width:40%; max-width:100%; height:auto;" />
  <img src="{{ 'assets/images/etapy_bud/zaw2.jpg' | relative_url }}" alt="zaw2" style="width:40%; max-width:100%; height:auto;" />
  <img src="{{ 'assets/images/etapy_bud/zaw3.jpg' | relative_url }}" alt="zaw3" style="width:60%; max-width:100%; height:auto;" />
</div>

#### 2) Belka nośna, czyli „most”
Kolejny element to belka, którą można nazwać mostem przednim lub tylnym. To ona trzyma po parze wózków kół i przenosi siły na ramę. Na jednym ze zdjęć rozłożone są części (elementy wykonane w technice druku 3D): czarny element belki, biała płyta – to pierwszy segment dolnej płyty nośnej – oraz pierwsza, czarna sekcja górnej płyty. Obok leżą pasek i zębatki. Ten zestaw pozwala przenieść ruch serwa skrętu na obie zwrotnice jednocześnie. Napęd skrętu działa prosto: serwo porusza zębatką, ta porusza paskiem, a pasek obraca obie kolumny z osiami kół w przeciwnych kierunkach, tak aby koła skręcały się zgodnie.

Na zdjęciu zmontowanej belki widać, że już na tym etapie łączymy ją z fragmentami płyty dolnej i górnej. To wygodne, bo łatwiej zachować równoległość i proste prowadzenie paska. Gniazda pod łożyska i dystanse są „zdrukowane” od razu, więc elementy pasują na wcisk i wystarczy je skręcić śrubami, aby całość była sztywna. Co jest ważne - tak jak poprzednio należy wykonać nacięcia na osi w celu bezluzowego zablokowania zębatek. 
Ilustracje: 

<div style="display:flex; gap:12px; flex-wrap:wrap;">
  <img src="{{ 'assets/images/etapy_bud/Belka1.jpg' | relative_url }}" alt="Belka1" style="width:40%; max-width:100%; height:auto;" />
  <img src="{{ 'assets/images/etapy_bud/Belka2.jpg' | relative_url }}" alt="Belka2" style="width:40%; max-width:100%; height:auto;" />
</div>

#### 3) Podwozie i nadwozie – trzyczęściowa rama
Rama składa się z trzech dużych wydruków 3D na dół i trzech na górę. Elementy mają wpusty, które działają jak zatrzaski prowadzące i wymuszają właściwe ułożenie. Po wstępnym „kliknięciu” skręcamy wszystko śrubami. Taka konstrukcja jest lekka, a jednocześnie sztywna. Dolna płyta tworzy podłogę na baterię i konwertery zasilania Vitron Orion. Na zdjęciu z wnętrza widać, że bateria siedzi na białej podstawie po prawej stronie, a obok niej po lewej, stoją dwa konwertery, w kolorze niebieskim. Wtyczki XT90 są łatwo dostępne z boku, więc podłączanie i odłączanie zasilania jest szybkie.

Górna płyta to nasz „stół” pod elektronikę. Wydruk ma przeloty na śruby, gniazda pod dystanse oraz wycięcia na przewody. Wysokość dystansów jest dobrana tak, aby między piętrami zostało miejsce na ruch zawieszenia i na przewody od kół, a jednocześnie żeby środek ciężkości był jak najniżej. Po złożeniu dwóch mostów z kołami do dolnej płyty i postawieniu na dystansach górnej płyty otrzymuje sztywną, dwupiętrową konstrukcję.
Ilustracje:

<img src="{{ 'assets/images/etapy_bud/Podw1.JPG' | relative_url }}" alt="Podw1" style="width:60%; max-width:100%; height:auto;" />

#### 4) Elektronika na pokładzie
Na górze umieszczam komputer Jetson Orin Nano (master) w standardowym radiatorze z wentylatorem. Obok są dwa moduły oparte o ESP32: jeden działa jako Hub Motors i obsługuje silniki DDSM400 oraz odczyt ich enkoderów, drugi to Bus Servo, który steruje serwami ST3215 i czyta ich pozycję z enkoderów. Każdy moduł ma własny, drukowany uchwyt, tak aby złącza były skierowane do środka i żeby przewody miały krótki, prosty przebieg. W narożach płyty znajdują się białe „klocki” – to obudowy serw ST3215 odpowiedzialnych za skręt osi. Czerwony główny włącznik jest na wierzchu, w zasięgu ręki. Z tyłu mam wygodne gniazdo umożliwiające podłączenie czytnika pomiaru napięcia.
Ilustracje:

<img src="{{ 'assets/images/etapy_bud/Elektr1.JPG' | relative_url }}" alt="Elektr1" style="width:60%; max-width:100%; height:auto;" />

#### 5) Układ zasilania
Projektując układ zasilania zaczynam od wymagań napięciowych. Jetson potrzebuje 9–20 V. Silniki DDSM400 (Hub Motors) pracują w zakresie 9–28 V. Serwa z magistrali Bus Servo wymagają 9–12.6 V. Do tego dochodzi zapotrzebowanie na moc: Jetson co najmniej 25 W, cztery silniki po około 25 W każdy, dwa serwa również po około 25 W. W krótkich szczytach cały układ może poprosić nawet o około 175 W. Jetson jest przy tym bardzo wrażliwy na zapady napięcia – przy spadkach potrafi się resetować albo „zgubić” kontrolery. Dlatego zasilanie musi być odporne i stabilne.

Wybrałem układ jednobateryjny. To najprostsze rozwiązanie: jeden akumulator LiPo 6S, około 24 V, 11 Ah, z wysokim dopuszczalnym prądem rozładowania 110C. Jedna bateria to mniej przewodów, mniej złącz i prostsze ładowanie. W tym wariancie Hub Motors zasilam bezpośrednio z baterii 24 V, natomiast Jetsona i magistralę serw zasilam przez stabilne przemysłowe przetwornice buck 12 V o dużej mocy. Zastosowałem Victron Orion 18–36 V / 12 V, 20 A (240 W) – po jednej sztuce dla Jetsona i dla Bus Servo. Dzięki temu komputer i serwa dostają czyste, trzymane na sztywno 12 V, niezależnie od tego, co dzieje się na linii napędu.

Szczegóły połączeń oraz zastosowane zabezpieczenia prądowe (główny bezpiecznik przy baterii, bezpieczniki na wyjściach i podział masy na wspólnej szynie) pokazuje dołączony schemat.
Ilustracje:

<img src="{{ 'assets/images/etapy_bud/Schemat zasilania.png' | relative_url }}" alt="Schemat zasilania" style="width:60%; max-width:100%; height:auto;" />

#### 6) Komunikacja
Sterowanie zaczynamy od kontrolera Bluetooth. Sygnał z pada trafia do małego odbiornika USB, który wpinamy w Jetsona. Jetson widzi go jak zwykłą klawiaturę/joystick i czyta przyciski oraz drążki. Z Jetsona wychodzą dwa kable USB – każdy do innego ESP32. Pierwszy idzie do ESP32 DDSM Driver HAT, który obsługuje cztery silniki DDSM400 (napęd). Drugi idzie do ESP32 Servo Driver, który steruje dwoma serwami ST3215 (skręt). Każdy ESP jest osobnym urządzeniem szeregowym i dostaje swój port, np. /dev/ttyUSB0 i /dev/ttyUSB1. Dzięki temu nie ma konfliktów – program na Jetsonie otwiera dwa niezależne połączenia.

Zasilanie silników i serw jest oddzielne od zasilania Jetsona, ale wszystkie urządzenia mają wspólną masę z ESP. To ważne, bo sygnały wtedy „wiedzą”, gdzie jest zero volt i nie pojawiają się błędy transmisji.

Pełny dupleks – co to daje

Komunikacja jest pełnodupleksowa. Oznacza to, że jednocześnie wysyłamy komendy i odbieramy dane zwrotne. W praktyce Jetson podaje polecenia: ustaw prędkość kół, ustaw kąt skrętu. W tym samym czasie ESP odsyła feedback: aktualne prędkości, kąty, prądy, temperatury. Nic na nic nie czeka – wysłanie komendy nie blokuje odbioru, a odbiór nie blokuje wysyłania. Operator może nacisnąć skręt w lewo, a dokładnie w tej samej chwili Jetson odbiera nowy pomiar prędkości z kół. Dzięki temu reakcje są płynne, a sterowanie „miękkie” i przewidywalne.

Co robi ESP

Na ESP działają dwa niezależne bufory UART: jeden do odbioru, drugi do nadawania. W pętli urządzenie bez przerwy sprawdza, czy przyszła komenda, wykonuje ją i w stałym rytmie (np. co 20 ms, czyli 50 Hz) pakuje i wysyła paczkę feedbacku. Format wiadomości jest lekki i stały, żeby parsowanie było szybkie.

Co robi Jetson

Na Jetsonie w Pythonie uruchamiamy dwa wątki na każdy port:


- wątek odbioru otwiera /dev/ttyUSBx i tylko czyta linie (readline), od razu je parsuje i odkłada do kolejki z danymi,

- wątek wysyłania nasłuchuje zdarzeń z pada i wysyła komendy, kiedy coś się zmieni (np. drążek ruszył się o więcej niż ustalony próg).

Odbiór nie ma sztucznych opóźnień – tempo nadaje ESP. Wysyłanie jest „na zdarzenia”, więc nie zapychamy łącza niepotrzebnymi powtórkami. Obie strony korzystają z kolejek/buforów, więc krótkie „czkawki” w systemie nie powodują gubienia danych.

Dlaczego to działa dobrze


- Dwa osobne kable USB i dwa osobne porty szeregowe upraszczają kod i eliminują kolizje.

- Oddzielne zasilanie napędu i serw od strony mocy oraz wspólna masa z ESP zapewniają czysty sygnał.

- Pełny dupleks utrzymuje stały strumień pomiarów, a jednocześnie reaguje natychmiast na sterowanie.

#### 7) Tryby pracy systemu
System ma kilka trybów, które przełączamy sprzętowo lub z poziomu aplikacji. Chodzi o to, aby w danym momencie robić tylko to, na co realnie starcza mocy Jetsona, i mieć jasne priorytety: albo jedziemy i logujemy, albo pokazujemy interfejs, albo uruchamiamy model.

Drive‑Log

To tryb do jazdy i zbierania danych. Prowadzimy ręcznie padem albo używamy prostej pętli sterowania (PID/MPC), która trzyma prędkość i kierunek. Nie uruchamiamy LLM, a analiza obrazu jest minimalna lub wyłączona. Najważniejsze jest stabilne logowanie: prędkości kół, kąty skrętu, prądy, pozycje serw, ewentualnie klatki z kamer i IMU. Dzięki temu logi mają równy rytm i nadają się do późniejszego treningu i do SLAM w postprocesie.

Autonomy‑Run

To jazda na wytrenowanej sieci. Model działa w wersji lekkiej, zoptymalizowanej na Jetsona (TensorRT, INT8). LLM jest wyłączony. Zbieramy telemetrię, ale w niższej rozdzielczości, żeby nie dławić GPU/CPU. Priorytetem jest płynna inferencja i bezpieczne sterowanie. Jeśli trzeba, ograniczamy częstotliwość zapisu klatek lub kompresujemy je mocniej.

HMI‑Only

Tryb pokazowy interfejsu. Uruchamiamy LLM i mowę (ASR/TTS), ale nie włączamy wizji ani SLAM. Komendy są wysokopoziomowe i bezpieczne, np. start, stop, zmiana trybu. Pojazd nie wykonuje złożonej autonomii – to sterowanie „rozumiejące polecenia”, a nie algorytm jazdy.

Safety Override

Nad wszystkim jest warstwa bezpieczeństwa. Mamy fizyczny E‑stop dostępny zawsze oraz watchdog, który pilnuje, czy połączenia żyją i czy pętle sterowania działają. Jeśli coś się zawiesi, system przechodzi do bezpiecznego stanu: odcięcie napędu, wyprostowanie kół, sygnalizacja.

Zbieranie danych w Drive‑Log

Kluczem jest czas. Ustalamy jedno źródło czasu: monotoniczny zegar systemowy na Jetsonie. ESP32 dołącza do każdego pakietu własny timestamp, a po stronie Jetsona robimy korekcję po odebraniu, żeby wyrównać opóźnienia. W metadanych zapisujemy odchyłkę (skew) i opóźnienie (latency), żeby później podczas łączenia strumieni (koła, serwa, IMU, kamera) móc je precyzyjnie zsynchronizować. Jeśli znaczniki nie pokryją się idealnie, dopuszczamy interpolację między próbkami.

Założenia wydajnościowe

Jetson nie pociągnie jednocześnie LLM i ciężkiej analizy obrazu z kamer przy pełnej szybkości. Dlatego rozdzielamy te zadania trybami. Jeśli już coś robimy równolegle podczas jazdy, to tylko klasyczne sterowanie (pad, PID/MPC) albo inference wytrenowanej, lekkiej sieci – wtedy LLM jest wyłączony. Dane do treningu i do SLAM liczymy potem na mocniejszej maszynie, z wygodnym czasem i bez ryzyka gubienia klatek w trakcie przejazdu

# Cele dydaktyczne
- Pokazać, że całą platformę da się zaprojektować w otwartym, chmurowym CAD (np. Onshape), a następnie wykonać potrzebne elementy na drukarce 3D.

- Przedstawić przejrzysty projekt zasilania: jedna bateria 6S, rozdział gałęzi, dwa konwertery 12 V dla Jetsona i serw, bezpośrednie 24 V dla Hub Motors, bezpieczniki i wspólna masa.

- Uporządkować założenia sterowania i komunikacji: dwa niezależne łącza USB do ESP32, pełny dupleks, separacja zasilania mocy i sygnału.

- Uświadomić konieczność zdefiniowania trybów pracy (Drive‑Log, Autonomy‑Run, HMI‑Only, Safety Override) przed rozpoczęciem implementacji oprogramowania.

# Co dalej
W kolejnym kroku wykorzystamy model kinematyczny Ackermanna do planowania trajektorii i do szacowania błędu pozycjonowania. Pokażemy, jak uwzględniać opóźnienia i niepewność pomiarów oraz jak aktualizować czas i wariancję na podstawie prawa propagacji błędu, tak aby logi z Drive‑Log przekładały się na stabilną, przewidywalną jazdę w Autonomy‑Run.