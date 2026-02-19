---
layout: post
title: "Kamera jako czujnik położenia"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Map fitting, co to jest i dlaczego to “nie działa”?<!--more-->


<!-- MathJax tylko dla tego wpisu -->
<!-- MathJax dla $…$, $$…$$ oraz \( … \), \[ … \] -->
<script>
  window.MathJax = {
    tex: {
      inlineMath: [['$', '$'], ['\\(', '\\)']],
      displayMath: [['$$', '$$'], ['\\[', '\\]']],
      processEscapes: true,
      processEnvironments: true
    },
    options: {
      skipHtmlTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
    }
  };
</script>

<script
  id="MathJax-script"
  async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js"
></script>


### 1) Problem
Zaczynam od obrazu przedstawionego na ry. 1, który na pierwszy rzut oka wygląda poprawnie. Widzimy trajektorię pojazdu i fragmenty pasów ruchu przeniesione do wspólnego, globalnego układu odniesienia. Każda klatka kamery została przekształcona do widoku z lotu ptaka (BEV), a następnie “położona” na mapie w oparciu o odometrię. Intuicyjnie wszystko się zgadza: kamera widzi podłogę, znamy pozycję pojazdu, więc wystarczy te obserwacje przesuwać i obracać zgodnie z ruchem.
Problem polega na tym, że to nie powinno działać. A jeśli działa — to tylko pozornie.

<img src="{{ 'assets/images/KameraCzujnik/surowa_traj.png' | relative_url }}" alt="surowa_traj" style="width:100%; max-width:100%; height:auto;" />


### 2) Od obrazu do świata
Kamera w każdej chwili obserwuje fragment rzeczywistości w swoim lokalnym układzie odniesienia. Po przekształceniu do BEV geometria staje się w przybliżeniu metryczna: odległości mają sens, kąty są zachowane, a perspektywa przestaje dominować. To bardzo kuszące — skoro mamy lokalny “plan podłogi”, to wystarczy go umieścić w układzie świata. I dokładnie w tym miejscu zaczynają się kłopoty.
Jeżeli składamy kolejne klatki wyłącznie na podstawie surowej odometrii, każda drobna niedokładność modelu ruchu — minimalny błąd kąta, niewielka asymetria skrętu, makro poślizg koła — powoduje, że kolejny fragment pasa odkładany jest w nieco złym miejscu. Te błędy nie znikają. One się sumują.
Po przejechaniu pętli mapa nie domyka się, a pas zaczyna “płynąć”. To nie jest błąd obrazu.
To jest konsekwencja niespójności transformacji między lokalnym układem kamery a układem świata — wymuszonej przez błędy trajektorii. Rys. 2 przedstawia błędnie złożoną mapę przy użyciu samej odometrii. 

<img src="{{ 'assets/images/CameraSensor/bledna_mapa.png' | relative_url }}" alt="ledna_mapa" style="width:100%; max-width:100%; height:auto;" />


### 3) Landmarki jako punkty stałe
Żeby zrozumieć, gdzie leży problem, potrzebny jest punkt odniesienia. A najlepiej kilka — i to takich, które na pewno się nie poruszają. Dlatego pojawiają się landmarki. Na podłodze leży mata z markerami ArUco. Mata zastosowana w eksperymencie z rozłożonymi na niej pasami ruchu i markerami jest przedstawiona na rys. 3.

<img src="{{ 'assets/images/CameraSensor/makieta.png' | relative_url }}" alt="makieta" style="width:100%; max-width:100%; height:auto;" />

Każdy marker ma unikalne ID, jest jednoznacznie rozpoznawalny i daje precyzyjnie wyznaczoną pozycję w obrazie. W kolejnych klatkach, po transformacji do BEV, zapisujemy położenie markerów w lokalnym układzie kamery (albo w “lokalnym BEV” związanym z pojazdem). Każda klatka daje nam więc dwie rzeczy:

-	fragment pasa ruchu,
-	współrzędne punktu środkowego markera, który w rzeczywistości jest nieruchomy.

<img src="{{ 'assets/images/CameraSensor/markery.png' | relative_url }}" alt="markery" style="width:100%; max-width:100%; height:auto;" />

Pozycje markerów zapisuje w odpowiedniej tablicy csv. Jeżeli po przekształceniu do układu świata ten sam marker pojawia się w różnych miejscach, wiemy jedno: to nie podłoga się rusza. To model ruchu przestaje być zgodny z rzeczywistością. Na rysunku 5 pokazano globalne pozycje tych samych landmarków obserwowanych w różnych momentach przejazdu. Landmarki o tym samym ID tworzą wyraźne, rozdzielone skupiska. Każde skupisko odpowiada innemu fragmentowi trajektorii. Różnice między nimi nie są losowe — odzwierciedlają systematyczny dryf odometrii oraz błędne założenia co do położenia i orientacji kamery względem pojazdu.

<img src="{{ 'assets/images/CameraSensor/surowe.png' | relative_url }}" alt="surowe" style="width:100%; max-width:100%; height:auto;" />

### 4) Etap I – Offline Map Fitting
W tym miejscu warto jasno powiedzieć, co robimę.
To nie jest jeszcze SLAM.
Jestem w Etapie I – Offline Map Fitting, który ma bardzo konkretne założenia:

- trajektoria pojazdu jest zamrożona i traktowana jako prawda,
-	nie wolno jej zmieniać,
-	estymujemy tylko:
-  -  parametry systemowe (ekstrynsykę kamery),
-  - 	położenia landmarków w jednej, wspólnej mapie.

Innymi słowy: szukamy najlepiej dopasowanej mapy świata, zakładając, że trajektoria jest idealna.
To kluczowe założenie. I — jak się za chwilę okaże — bardzo silne.

### 5) Minimalna formalizacja (żeby nie zgubić sensu)
W dalszym opisie użyję dwóch pojęć:
-	$C$ — “trajektoria” (kolejne pozycje pojazdu w czasie),
-	$G$ — “mapa” (globalny układ odniesienia i pozycje landmarków w tym układzie).

W chwili $k$ znamy (z odometrii) pozę pojazdu: 
$$
\mathbf{x}_k = (x_k, y_k, \psi_k).
$$
Z detekcji dostajemy pomiar landmarku: $\mathbf{z}_{k,j}$
w układzie lokalnym (BEV/kamery). Chcemy go przenieść do świata i porównać z globalną pozycją landmarku $\mathbf{p}_j$.
W praktyce (w 2D) sprowadza się to do składania transformacji:

-	najpierw “lokalny punkt z BEV”,
-	potem ekstrynsyka (stała transformacja między kamerą/BEV a bazą pojazdu),
-	potem poza pojazdu w świecie.

Można to zapisać zwięźle jako:

$$
\begin{aligned}
\hat{\mathbf{p}}_{k,j}^{G} = T_{G\leftarrow C}(\mathbf{x}_k)\ \circ\ T_{C}(\theta)\ \circ\ \mathbf{z}_{k,j}
\end{aligned}
$$

gdzie:
-	$T_{G\leftarrow C}(\mathbf{x}_k)$ wynika z zamrożonej trajektorii,
-	$T_{C}(\theta)$ to estymowana ekstrynsyka (np. przesunięcie i skręt kamery/BEV względem pojazdu),
-	$\mathbf{z}_{k,j}$ to pomiar landmarku w układzie lokalnym.

A cel Etapu I to minimalizacja rozrzutu obserwacji tego samego landmarku w świecie, przy stałej trajektorii:

$$
\min_{\theta,\{\mathbf{p}_j\}} \sum_{(k,j)\in\mathcal{O}} \left\| \hat{\mathbf{p}}_{k,j}^{G} - \mathbf{p}_j \right\|^2
$$

To są dwa wzory, które “spinają” cały tekst: pokazują, co liczymy i dlaczego.
W wersji “programistycznej” (czytelniejszą dla osób z robotyki), to ten sam łańcuch można zapisać tak:

```
landmark_in_G = pose_G_from_traj[k] ∘ T_extrinsic ∘ z_kj
```

gdzie:
-	```pose_G_from_traj[k]``` – zamrożona trajektoria (odometria),
-	```T_extrinsic``` – estymowana ekstrynsyka,
-	```z_kj``` – pomiar landmarku w układzie lokalnym (kamera/BEV; analogicznie może to być lidar).

### 6) Kalibracja ekstrynsyki, czyli kiedy geometria przestaje wystarczać
W projekcie CAD samochodu przyjąłem, że punkt G ("kotwica" kamery tzn. początek lokalnego układu współrzędnych związanego z BEV) znajduje się w określonej odległości od środka modelu kinematycznego pojazdu. Znamy rozstaw osi, znamy promień kół, znamy przybliżone położenie kamery względem konstrukcji. To naturalny punkt startowy. Tyle że model nigdy nie jest rzeczywistością.
Kamera może być minimalnie skręcona. Oś optyczna może nie być idealnie równoległa do osi pojazdu. A co ważniejsze — sam model ruchu jest kompromisem. Choć korzystamy z uproszczonego modelu Ackermanna, pojazd ma cztery koła skrętne sterowane parami. Koła wewnętrzne i zewnętrzne skręcają się pod tym samym kątem. Nie ma dyferencjału kątowego — jest tylko dyferencjał prędkościowy. W praktyce oznacza to jedno: rzeczywisty tor ruchu nie może idealnie odpowiadać modelowi. Pojawiają się poślizgi, sprężystość opon, asymetrie napędu. Tego nie da się „dokalibrować” jedną stałą.
Dlatego zamiast wierzyć geometrii, stosujemy kryterium obiektywne: minimalizujemy rozrzut globalnych pozycji tych samych landmarków. Szukamy takich parametrów transformacji, dla których landmarki w układzie świata są możliwie najbardziej zwarte.
Punkt G został wyznaczony projektowo jako L/2+R. Jednak analiza wariancji landmarków ujawniła,
że rzeczywiste położenie układu BEV różni się o od tej wartości. Projekt przestał być założeniem — stał się hipotezą, którą można weryfikować danymi. To moment, w którym projekt mechaniczny staje się elementem systemu pomiarowego, który można oceniać statystycznie. Efekty minimalizacji rozrzutu grup landmarków poprzez zmiany współrzędnych punku G przedstawia rys. 6. 


<img src="{{ 'assets/images/CameraSensor/ekstrynsyka.png' | relative_url }}" alt="ekstrynsyka" style="width:100%; max-width:100%; height:auto;" />

### 7) Algorytm "map fitting" działa świetnie. Mapa? – niekoniecznie
Po optymalizacji "map fitting" dzieje się coś pozornie idealnego. Koszt funkcji dopasowania grup landmarków do trajektorii gwałtownie spada. Landmarki układają się w zwarte skupiska. Algorytm nie widzi już sprzeczności — wszystko „pasuje”. Tylko, że gdy na tę mapę spojrzymy w odniesieniu do rzeczywistości, coś się nie zgadza. Landmarki są spójne względem trajektorii, ale nie względem świata. Fragmenty pasa ruchu są przesunięte. A różnice nie są losowe — pojawiają się dokładnie tam, gdzie ruch pojazdu był najbardziej wymagający. I to nie jest błąd optymalizacji. To jej logiczna konsekwencja. Dzieje się tak dlatego, że algorytm znajduje rozwiązanie matematycznie spójne — ale tylko względem przyjętej trajektorii (ponieważ ona z zalożenia jest "zamrożona" - prawdziwa i nie mogże być korygowana). Wynik działania "map fittigu" ukazuje rys. 7. 

<img src="{{ 'assets/images/CameraSensor/map_fitting.png' | relative_url }}" alt="map_fitting" style="width:100%; max-width:100%; height:auto;" />

Zielone i różowe punkty na wykresach nie są trajektorią. One pokazują, gdzie w globalnym układzie lądują lokalne obserwacje landmarków, gdy przeniesiemy je przez zamrożoną trajektorię.
Jeżeli trajektoria byłaby idealna, wszystkie obserwacje tego samego obiektu nałożyłyby się na siebie. Jeżeli nie jest — chmury zaczynają się wyginać, przesuwać i rozjeżdżać. Szczególnie wyraźnie widać to na ostrych zakrętach. W tych miejscach rzeczywisty promień skrętu jest większy niż wynikałoby to z modelu. Pojazd nie obraca się wokół jednego, idealnego środka chwilowego obrotu. Może to być efektem poślizgów. Model Ackermanna przestaje być dobrym przybliżeniem — a kamera bezlitośnie to ujawnia.

### 8) A jak to się ma do rzeczywistości?
Na rysunku 8 zestawiono trzy informacje: trajektorię odometryczną, globalne skupiska landmarków po map fittingu oraz rzeczywiste położenia markerów. Dla części landmarków (1–5) zgodność jest bardzo dobra, natomiast dla kolejnych (6–8) pojawiają się systematyczne przesunięcia. Od tego momentu cały układ zaczyna „odjeżdżać”. Różnice te korelują z ostrymi zakrętami, gdzie rzeczywisty ruch pojazdu odbiega od uproszczonego modelu kinematycznego.

<img src="{{ 'assets/images/CameraSensor/map_fitting2.png' | relative_url }}" alt="map_fitting2" style="width:100%; max-width:100%; height:auto;" />

Można się w tym miejscu zastanowić dlaczego map fitting „nie działa” i co z tego wynika?

Offline map fitting robi dokładnie to, co do niego należy:

-	nie poprawia trajektorii,
-	przesuwa landmarki tak, aby pasowały do przyjętego ruchu.

Jeżeli trajektoria jest błędna, mapa zostanie zdeformowana. Algorytm nie ma innego wyjścia.
I właśnie dlatego ten etap jest tak ważny dydaktycznie. On nie służy do „naprawiania mapy”. On służy do zdiagnozowania, gdzie i dlaczego model ruchu przestaje być zgodny z rzeczywistością. 

Na tym etapie pipeline jest spójny, ale ograniczony:

-	kamera daje lokalne obserwacje w BEV,
-	odometria zapewnia ciągłość ruchu,
-	transformacja między układami jest skalibrowana,
-	landmarki pozwalają mierzyć rozjazd pętli.

Ale jedna rzecz jest już jasna: nie da się zbudować poprawnej mapy świata, jeżeli nie pozwolimy trajektorii się zmieniać. I to jest dokładnie moment, w którym Etap I musi się skończyć.

### 9) co dalej
Dopiero kolejne kroki — korekta trajektorii, pełny SLAM i modelowanie poślizgu — mają sens. Ale to już osobna historia. Spróbuje to zrobić póżniej. Na razie kamera zrobiła coś znacznie ważniejszego niż „zobaczenie pasa”. Stała się czujnikiem niespójności modelu ruchu. A to jest fundament każdego sensownego systemu lokalizacji.

