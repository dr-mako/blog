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
Zaczynam od obrazu przedstawionego na rys. 1, który na pierwszy rzut oka wygląda poprawnie. Widzimy trajektorię pojazdu i dwa fragmenty pasów ruchu przeniesione do wspólnego, globalnego układu odniesienia. Każda klatka kamery została przekształcona do widoku z lotu ptaka (BEV), a następnie “położona” na mapie w oparciu o odometrię. Intuicyjnie wszystko się zgadza: kamera widzi podłogę, znamy pozycję pojazdu, więc wystarczy te obserwacje przesuwać i obracać zgodnie z ruchem.
Problem polega na tym, że to nie powinno działać. A jeśli działa — to tylko pozornie.

<img src="{{ 'assets/images/CameraSensor/surowa_traj.png' | relative_url }}" alt="surowa_traj" style="width:100%; max-width:100%; height:auto;" />


### 2) Od obrazu do świata
Kamera w każdej chwili obserwuje fragment rzeczywistości w swoim lokalnym układzie odniesienia. Po przekształceniu do BEV geometria staje się w przybliżeniu metryczna: odległości mają sens, kąty są zachowane, a perspektywa przestaje dominować. To bardzo kuszące — skoro mamy lokalny “plan podłogi”, to wystarczy go umieścić w układzie świata. I dokładnie w tym miejscu zaczynają się kłopoty.
Jeżeli składamy kolejne klatki wyłącznie na podstawie surowej odometrii, każda drobna niedokładność modelu ruchu — minimalny błąd kąta, niewielka asymetria skrętu, makro poślizg koła — powoduje, że kolejny fragment pasa odkładany jest w nieco złym miejscu. Te błędy nie znikają. One się sumują.
Po przejechaniu pętli mapa nie domyka się, a pas zaczyna “płynąć”. To nie jest błąd obrazu.
To jest konsekwencja niespójności transformacji między lokalnym układem kamery a układem świata — wymuszonej przez błędy trajektorii. Rys. 2 przedstawia kilka wybranych klatek obrazów z całego przejazdu złożonych w opisany powyżej sposób. Pasy ruchu rozjeżdzają się - szczególnie w okolicy ostrych zakrętów. Wynik ten pokazuje, że przy użyciu samej odometrii nie da się z obrazów BEV prawidłowo złożyć mapy. 

<img src="{{ 'assets/images/CameraSensor/bledna_mapa.png' | relative_url }}" alt="bledna_mapa" style="width:100%; max-width:100%; height:auto;" />


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
W tym miejscu warto jasno powiedzieć, co robimy. To nie jest jeszcze SLAM. W Etapie I – Offline Map Fitting przyjmuję następujące założenia:

- trajektoria pojazdu jest zamrożona i nie podlega korekcji,
- współrzędne lokalne landmarków otrzymuję z wcześniej wyznaczonego przekształcenia BEV,
- transformacja związana z geometrią kamery i pojazdu jest ustalona i nie jest optymalizowana,
- dla każdego ID wyznaczam jedną pozycję landmarku w układzie globalnym oraz oceniam rozrzut jego obserwacji.

Innymi słowy: sprawdzam, jak spójna jest mapa zbudowana z obserwacji kamery przy założonej trajektorii odometrycznej. Jeżeli trajektoria zawiera błędy, będą one widoczne w rozrzucie globalnych obserwacji nieruchomych landmarków.

### 5) Minimalna formalizacja (żeby nie zgubić sensu)

W dalszym opisie użyję dwóch pojęć:

- $C$ – „trajektoria”, czyli kolejne pozycje i orientacje pojazdu w czasie,

- $G$ – „mapa”, czyli globalny układ odniesienia i pozycje landmarków w tym układzie.

W chwili $k$ znamy z odometrii pozę pojazdu:

$$
[
\mathbf{x}_k=(x_k,y_k,\varphi_k)
]
$$

Z detekcji markera otrzymujemy jego położenie na obrazie, a następnie, po przekształceniu do metrycznego BEV, współrzędne $(X_{\mathrm{BEV}},Y_{\mathrm{BEV}})$.
W rozpatrywanym przypadku zależność pomiędzy układem BEV a lokalnym układem pojazdu jest znana. Wynika ona z pomiaru geometrii stanowiska, w szczególności położenia planszy kalibracyjnej względem środka geometrycznego samochodu. Zależność tę uwzględniam już podczas wyznaczania przekształcenia obrazu do BEV. Otrzymane współrzędne metryczne są więc odniesione bezpośrednio do pojazdu, bez konieczności dodatkowego wyznaczania parametrów zewnętrznych kamery na etapie map fittingu.
Trzeba natomiast uwzględnić różnicę w orientacji osi obu układów. W układzie BEV oś $Y_{\mathrm{BEV}}$ wskazuje kierunek jazdy, a oś $X_{\mathrm{BEV}}$ kierunek poprzeczny. W lokalnym układzie odometrii oś $x$ jest skierowana do przodu, natomiast oś $y$ w lewo. 
Tą sytuację przedstawia rys:

<img src="{{ 'assets/images/CameraSensor/ekstrynsyka.png' | relative_url }}" alt="ekstrynsyka" style="width:100%; max-width:100%; height:auto;" />

W tym przypadku współrzędnych ma zatem postać:

$$
[
\mathbf{z}{k,j}^{\mathrm{local}}=
\begin{bmatrix}
Y{\mathrm{BEV}}\
-X_{\mathrm{BEV}}
\end{bmatrix}
]
$$

Tak otrzymany pomiar landmarku jest już wyrażony względem pojazdu. Aby przenieść go do układu globalnego, wystarczy obrócić go o kąt orientacji pojazdu i dodać jego aktualną pozycję:

$$
[
\begin{bmatrix}
x_k\
y_k
\end{bmatrix}
+
R(\varphi_k)\mathbf{z}_{k,j}^{\mathrm{local}}
]
$$

gdzie:

- $R(\varphi_k)$ – macierz obrotu wynikająca z orientacji pojazdu,

- $\mathbf{z}_{k,j}^{\mathrm{local}}$ – lokalny pomiar landmarku $j$ w chwili $k$,

- $\hat{\mathbf{p}}_{k,j}^{G}$ – globalna pozycja tego landmarku obliczona z pojedynczej obserwacji.

Ponieważ ten sam nieruchomy marker może być obserwowany w wielu klatkach, otrzymujemy zbiór jego globalnych pozycji. Gdyby odometria i pomiary kamery były idealne, wszystkie te punkty pokrywałyby się.

W praktyce tworzą skupisko. Jego środek wyznaczam jako średnią obserwacji:

$$
[
\bar{\mathbf{p}}j^G =
\frac{1}{N_j}
\sum{k=1}^{N_j}\hat{\mathbf{p}}_{k,j}^{G}
]
$$

a rozrzut opisuję błędem RMS:

$$
[
{
\bar{\mathbf{p}}_j^G
\right|^2
}
]
$$

Otrzymujemy w ten sposób jedną reprezentatywną pozycję każdego landmarku oraz informację o spójności jego obserwacji.

Warto zwrócić uwagę na ograniczenie tego podejścia. Trajektoria pozostaje zamrożona, a algorytm nie koryguje ani pozy pojazdu, ani parametrów transformacji kamery. Jeżeli odometria dryfuje, wyznaczone pozycje landmarków również będą obciążone tym błędem.

To właśnie chcę sprawdzić na tym etapie: czy obserwacje tego samego nieruchomego punktu są ze sobą zgodne, gdy przyjmujemy trajektorię odometryczną za ustaloną.

W wersji programistycznej cały proces można zapisać znacznie prościej:

```
marker_local_x = Y_BEV
marker_local_y = -X_BEV

marker_global = (
    vehicle_position
    + R(phi) @ marker_local
)
```

Następnie dla każdego ID markera obliczam centroid jego globalnych obserwacji i RMS.
Nie szukam zatem ekstrynsyki, która sztucznie zmniejszy rozrzut punktów. Ten rozrzut jest teraz wynikiem eksperymentu i informacją o niespójności całego łańcucha pomiarowego.


### 7) Algorytm "map fitting" działa świetnie. Mapa? – niekoniecznie
Po optymalizacji "map fitting" dzieje się coś pozornie idealnego. Koszt funkcji dopasowania grup landmarków do trajektorii gwałtownie spada. Landmarki układają się w zwarte skupiska. Algorytm nie widzi już sprzeczności — wszystko „pasuje”. Tylko, że gdy na tę mapę spojrzymy w odniesieniu do rzeczywistości, coś się nie zgadza. Landmarki są spójne względem trajektorii, ale nie względem świata. Fragmenty pasa ruchu są przesunięte. A różnice nie są losowe — pojawiają się dokładnie tam, gdzie ruch pojazdu był najbardziej wymagający. I to nie jest błąd optymalizacji. To jej logiczna konsekwencja. Dzieje się tak dlatego, że algorytm znajduje rozwiązanie matematycznie spójne — ale tylko względem przyjętej trajektorii (ponieważ ona z zalożenia jest "zamrożona" - prawdziwa i nie może być korygowana). Wynik działania "map fittigu" ukazuje rys. 7. 

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

Algorytm znajduje rozwiązanie matematycznie poprawne, lecz fizycznie błędne.
Koszt maleje nie dlatego, że świat został dobrze opisany, ale dlatego, że mapa została zdeformowana tak, aby zgadzała się z błędną trajektorią.

-	kamera daje lokalne obserwacje w BEV,
-	odometria zapewnia ciągłość ruchu,
-	transformacja między układami jest skalibrowana,
-	landmarki pozwalają mierzyć rozjazd pętli.

Ale jedna rzecz jest już jasna: nie da się zbudować poprawnej mapy świata, jeżeli nie pozwolimy trajektorii się zmieniać. I to jest dokładnie moment, w którym Etap I musi się skończyć. Można go podsymować następującym sformułowaniem: "Best-fit map of landmarks assuming the trajectory is ground truth" czyli "najlepiej dopasowana mapa punktów orientacyjnych, przy założeniu, że trajektoria jest zgodna z rzeczywistością.

### 9) co dalej
Dopiero kolejne kroki — korekta trajektorii, pełny SLAM i modelowanie poślizgu — mają sens. Ale to już osobna historia. Spróbuje to zrobić później. Na razie kamera zrobiła coś znacznie ważniejszego niż „zobaczenie pasa”. Stała się czujnikiem niespójności modelu ruchu. A to jest fundament każdego sensownego systemu lokalizacji.

