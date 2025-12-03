---
layout: post
title: "Model pojazdu do badań nad autonomią"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---
# O czym będzie ten cykl.  <!--more-->
## Koncepcja projektu
Buduję model pojazdu, który ma jeździć po:

- torze PRT z laboratoriów Wydziału Transportu,
- makietach drogowych (miasto, skrzyżowania),
- prostych scenariuszach magazynowych.

To jedna wspólna platforma do ćwiczenia sterowania, pomiaru, estymacji stanu, percepcji i elementów AI. Będę pokazywał rzeczy działające, ale też ograniczenia i błędy.

## Co już jest
Na dziś mam gotowe podwozie samochodu z czterema kołami napędzanymi niezależnie — to tzw. napęd bezpośredni, z silnikami umieszczonymi w kołach. Skręt realizuję dwoma serwami: jednym dla pary kół przednich i jednym dla pary tylnych. Każda para kół jest sprzężona paskiem rozrządu. Taki układ jest sztywny i pozbawiony luzów, ale ma tę wadę, że koła skręcają się pod identycznym kątem. W efekcie promienie zataczane przez koła nie przecinają się w jednym wspólnym środku obrotu. Zastosowany napęd pozwala programowo wprowadzić wirtualny dyferencjał prędkościowy, ale nie umożliwia dyferencjału kątowego. Używam silników DDSM400 oraz serw ST3215 z wbudowanymi enkoderami magnetycznymi. Dzięki temu mogę stale kontrolować prędkość i kurs pojazdu, a odpowiednie filtry pozwalają na bieżąco śledzić jego trajektorię. Do sterowania i zapisu logów wykorzystuję komputer Jetson Orin Nano (wersja Super), który umożliwia uruchomienie lokalnego, małego modelu językowego. Komputer przyjmuje komendy głosowe i może zaplanować trasę przejazdu, a warstwę wykonawczą realizuje serwer w Pythonie. 

<img src="{{ 'assets/images/Wprowadzenie/Wprowadzenie.JPG' | relative_url }}" alt="Wprowadzenie" style="width:60%; max-width:100%; height:auto;" />

# Co planuję w najbliższych wpisach

#### 1) Sterowanie i geometria
Przedstawię bardzo prosty „rowerowy” model Ackermanna dla szczególnego przypadku dwóch osi skrętnych sterowanych jednym, wspólnym kątem skrętu (wszystkie koła skręcają się identycznie — przednia i tylna para w odbiciu lustrzanym, tj. naprzemiennie). Pokażę wzory pozwalające obliczyć promienie zataczania oraz opracować dyferencjał prędkościowy i kątowy. Wyjaśnię, dlaczego identyczne kąty skrętu w parach kół prowadzą do poślizgów w zakręcie. W tym punkcie przedstawię także równania ruchu mojego modelu.

#### 2) Odometria. Pomiary i estymacja
Pokażę, jak wykorzystuję pomiary prędkości oraz kąta skrętu do wyznaczania trajektorii ruchu. Zastosuję filtr Kalmana w dwóch krokach: predykcję (aktualizacja w czasie na podstawie modelu, z użyciem prawa propagacji błędu) oraz korekcję (aktualizacja na podstawie pomiaru, w ujęciu bayesowskim). Przedstawię wyniki pierwszych testów i omówię napotkane problemy.

#### 3) Percepcja i prowadzenie po linii
Pokażę metodę optycznego wykrywania pasów ruchu opartą na dwóch elementach: detekcji krawędzi (Canny/Sobel) oraz rzutowaniu „bird’s‑eye view” (z lotu ptaka). To podejście bywa wykorzystywane do prowadzenia pojazdów w magazynach. Zaprezentuję też prosty regulator podążania wzdłuż pasa drogi ograniczonego krawędziami.

#### 4) Sieci neuronowe i „end-to-end”
Pokażę wyzwania związane z dostrojeniem sieci AlexNet do zadania automatycznego utrzymywania kursu. Zbiór danych treningowych zbuduję z wielu przejazdów wykonanych prostym regulatorem opisanym wcześniej. Przeprowadzę trening i zaprezentuję działanie na torach ułożonych inaczej niż w danych uczących. Sprawdzę odporność na zakłócenia: „zanieczyszczenia” pasa ruchu, uszkodzenia krawędzi toru, a nawet brak jednej krawędzi.

#### 5) SLAM z jednej kamery
Zbuduję mapę otoczenia pojazdu na podstawie zdjęć rejestrowanych wzdłuż trasy przez kamerę zamontowaną na pojeździe. Składanie obrazów (image stitching) zrealizuję z użyciem metod cząstek, wykorzystując dodatkowo estymowane położenie i kurs pojazdu. Dzięki temu powstanie spójna mapa bez wsparcia GPS.

#### 6) LLM w pętli sterowania
Wykorzystam mały model językowy (Gemma 3 uruchomiona lokalnie na komputerze „master”) do planowania trasy. Z modelem komunikuję się głosowo, a właściwe sterowanie pojazdem realizuje Python. To demonstrator: „zastosowanie małego LLM na urządzeniu brzegowym do sterowania procesem transportowym”. Moim zdaniem ta technologia ma szansę stać się ważnym kierunkiem rozwoju systemów transportowych.

# Cele dydaktyczne
- Pokazać, jak uproszczenia modelu i błędy sterowania wpływają na trajektorię.
- Zademonstrować rolę filtru Kalmana w korygowaniu pomiarów.
- Porównać prowadzenie klasyczne (regulatory, przetwarzanie obrazu) z sieciami głębokimi.
- Zbudować minimalny przykład SLAM na jednej kamerze.
- Otworzyć dyskusję o sensownych zastosowaniach małych LLM w sterowaniu.

<img src="{{ 'assets/images/cwiczenie1/TorCw1.JPG' | relative_url }}" alt="TorCw1" style="width:75%; max-width:100%; height:auto;" />

# Co dalej
W następnym wpisie: etapy budowy platformy modelu — pokażę listę elementów, gdzie je kupić oraz jakie wydruki 3D przygotowałem, żeby złożyć prototyp.