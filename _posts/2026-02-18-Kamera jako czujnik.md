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
Żeby zrozumieć, gdzie leży problem, potrzebny jest punkt odniesienia. A najlepiej kilka, i to takich, które na pewno się nie poruszają. Dlatego pojawiają się landmarki. Na podłodze leży mata z markerami ArUco. Matę zastosowaną w eksperymencie, z rozłożonymi na niej pasami ruchu i markerami, przedstawiono na rys. 3.

<img src="{{ 'assets/images/CameraSensor/makieta.png' | relative_url }}" alt="makieta" style="width:100%; max-width:100%; height:auto;" />

Każdy marker ma unikalny identyfikator (ID), dzięki czemu można go jednoznacznie rozpoznać w kolejnych obrazach. Kamera pozwala wyznaczyć jego położenie w pikselach, a przekształcenie do metrycznego BEV umożliwia określenie współrzędnych środka markera względem pojazdu.
W ten sposób każda klatka dostarcza informacji o lokalnym położeniu obserwowanego landmarku. Sam marker pozostaje nieruchomy, natomiast samochód zmienia swoje położenie i orientację. Ta własność pozwala później wykorzystać powtarzające się obserwacje tych samych punktów do oceny spójności mapy.

<img src="{{ 'assets/images/CameraSensor/markery.png' | relative_url }}" alt="markery" style="width:100%; max-width:100%; height:auto;" />

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
\mathrm{RMS}_j =
\sqrt{
\frac{1}{N_j}
\sum_{k=1}^{N_j}
\left\|
\hat{\mathbf{p}}_{k,j}^{G}
-
\bar{\mathbf{p}}_j^G
\right\|^2
}
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


### 7) Ten sam landmark, różne pozycje na mapie

Pozycje markerów zapisuję w odpowiedniej tabeli CSV. Jeżeli po przekształceniu do układu globalnego ten sam nieruchomy marker pojawia się w różnych miejscach, oznacza to niespójność pomiarów. To nie podłoga się porusza. Problem leży w sposobie wyznaczania położenia pojazdu lub przeliczania obserwacji z kamery.
Na rys. 6 pokazano globalne pozycje tych samych landmarków obserwowanych w różnych momentach przejazdu. Landmarki o tym samym ID tworzą skupiska punktów, które nie zawsze pokrywają się ze sobą. Ich rozrzut nie musi być przypadkowy. Może wynikać z narastającego dryfu odometrii, szczególnie podczas ostrych zakrętów, ale również z niedokładności pomiarów w obrazie i przekształcenia BEV.

To właśnie ta niespójność jest przedmiotem dalszej analizy. Przy zamrożonej trajektorii map fitting pozwala wyznaczyć reprezentatywne pozycje landmarków i ocenić rozrzut ich obserwacji, ale nie usuwa przyczyn powstałych błędów.

<img src="{{ 'assets/images/CameraSensor/surowe.png' | relative_url }}" alt="Globalne pozycje obserwacji landmarków przy zamrożonej trajektorii odometrycznej" style="width:100%; max-width:100%; height:auto;" />

Dla każdego markera wyznaczam centroid globalnych obserwacji oraz błąd RMS opisujący ich rozrzut. Wyniki dla pierwszego przejazdu zestawiono w tabeli 1. Uwzględniono markery, dla których zarejestrowano co najmniej pięć obserwacji.


**Tabela 1. Estymowane pozycje globalne landmarków oraz RMS ich obserwacji przy zamrożonej trajektorii odometrycznej.**

| ID markera | N_j | x_G [m] | y_G [m] | RMS [mm] |
|:---:|---:|---:|---:|---:|
| 0  | 43 | 0.293 | 0.060 | 24.4 |
| 1  | 85 | 0.327 | 0.002 | 6.3 |
| 2  | 75 | 0.848 | 0.014 | 16.9 |
| 3  | 82 | 1.193 | -0.242 | 34.7 |
| 5  | 51 | 1.008 | -1.160 | 30.0 |
| 7  | 12 | 0.651 | -0.483 | 16.7 |
| 8  | 33 | 0.546 | -1.051 | 33.5 |
| 9  | 53 | 0.143 | -1.181 | 52.5 |
| 10 | 41 | -0.085 | -0.272 | 24.9 |
| 11 | 38 | -0.113 | -0.829 | 32.5 |

*W tabeli uwzględniono markery z co najmniej pięcioma obserwacjami. Średni RMS wynosi 27,25 mm.*


Rozrzut nie jest jednakowy. Największą spójność wykazują obserwacje markera ID 1, dla którego RMS wynosi 6,3 mm. Z kolei dla markera ID 9 otrzymano 52,5 mm. Średni RMS, obliczony dla dziesięciu uwzględnionych landmarków, wynosi 27,25 mm.
Wyznaczone centroidy są reprezentatywnymi pozycjami landmarków przy przyjętej trajektorii odometrycznej. Nie oznacza to jednak, że odpowiadają ich rzeczywistemu położeniu. Nawet niewielki RMS może występować wtedy, gdy wszystkie obserwacje danego markera są przesunięte w podobny sposób. Dlatego spójność wewnętrzną mapy trzeba odróżnić od jej zgodności z rzeczywistą geometrią stanowiska.

### 8) Czy geometria trasy wpływa na spójność mapy?

W pierwszym eksperymencie samochód poruszał się po stosunkowo krótkiej trasie z ostrymi zakrętami. Rozrzut globalnych obserwacji landmarków był wyraźny, a średni RMS wyniósł 27,25 mm. Największe różnice pojawiały się w obszarach związanych z intensywnymi manewrami.
Postanowiłem sprawdzić, jak zmienią się wyniki, gdy samochód będzie poruszał się po innej trasie. W drugim eksperymencie wykorzystałem dłuższy tor o łagodniejszych zakrętach. Sposób przetwarzania danych pozostał taki sam: obserwacje markerów przeliczam przez metryczne BEV do układu pojazdu, a następnie przenoszę do układu globalnego na podstawie odometrii. Trajektoria nadal pozostaje zamrożona. Na rys. 7 przedstawiono trajektorie dwóch przejazdów oraz globalne obserwacje landmarków dla drugiego eksperymentu.

<img src="{{ 'assets/images/CameraSensor/Przejazd 2 przed.png' | relative_url }}" alt="Przejazd 2 przed" style="width:100%; max-width:100%; height:auto;" />

Różnica jest wyraźna. Obserwacje tych samych markerów tworzą znacznie bardziej zwarte skupiska. W drugim eksperymencie wartości RMS dla większości landmarków mieszczą się w granicach kilkunastu milimetrów, a największa z przedstawionych w zestawieniu wynosi 15,3 mm.
To interesujący wynik. Pomimo zwiększenia długości trasy spójność obserwacji uległa poprawie. Sugeruje to, że geometria wykonywanych manewrów może mieć większe znaczenie niż sama długość przejazdu.
Można to powiązać z ograniczeniami przyjętego modelu ruchu. Podczas ostrych zakrętów rzeczywista kinematyka samochodu może silniej odbiegać od modelu wykorzystywanego w odometrii. Pojawiają się poślizgi kół, a niewielkie błędy orientacji przekładają się na przesunięcia obserwowanych landmarków w układzie globalnym.
Nie oznacza to jednak, że sam kształt toru odpowiada za całą zaobserwowaną różnicę. Na wyniki mogą wpływać także prędkość jazdy, jakość detekcji markerów i dokładność przekształcenia BEV. Do jednoznacznego określenia wpływu geometrii trasy potrzebne byłyby dodatkowe eksperymenty.
Porównanie pokazuje natomiast istotną własność całego układu: spójność mapy zależy od warunków ruchu pojazdu. Ten sam algorytm, wykorzystujący ten sam model kamery i niezmienioną metodę map fittingu, może dawać wyraźnie różne wyniki na dwóch trasach.
To ważna obserwacja przed przejściem do SLAM-u. Jeśli błędy odometrii zależą od wykonywanych manewrów, korekcja trajektorii powinna wykorzystywać informacje z ponownych obserwacji nieruchomych landmarków, szczególnie tam, gdzie model ruchu jest najmniej dokładny.

### 9) Jak druga pętla ujawnia dryf odometrii?
W obu eksperymentach samochód wykonywał dwa przejazdy po tej samej trasie. W pierwszym przypadku tor miał ostre zakręty, w drugim był dłuższy, a zakręty były łagodniejsze. Pozwala to porównać zachowanie całego układu pomiarowego w dwóch różnych warunkach ruchu.
Na rys. 8 zestawiono trajektorie odometryczne oraz globalne obserwacje landmarków dla obu eksperymentów. Linie ciągłe i przerywane przedstawiają kolejne przejazdy, natomiast kolory punktów odpowiadają identyfikatorom markerów.

<div style="display:flex; gap:10px; width:100%; align-items:center;">

  <img
    src="{{ 'assets/images/CameraSensor/Przejazd 1 po.png' | relative_url }}"
    alt="Przejazd 1"
    style="flex:1 1 0; width:100%; min-width:0; height:auto;"
  />

  <img
    src="{{ 'assets/images/CameraSensor/Przejazd 2 po.png' | relative_url }}"
    alt="Przejazd 2"
    style="flex:1 1 0; width:100%; min-width:0; height:auto;"
  />

</div>

Markery są nieruchome, dlatego ich rzeczywiste położenia nie zmieniają się pomiędzy przejazdami. Gdyby trajektoria odometryczna i pomiar z kamery były idealne, obserwacje tego samego markera trafiałyby w jedno miejsce na mapie, niezależnie od tego, w której pętli zostały zarejestrowane.
Wyniki pokazują jednak różnice. Na trasie z ostrymi zakrętami obserwacje tych samych markerów są wyraźnie rozdzielone pomiędzy przejazdami. Na dłuższej trasie z łagodniejszymi zakrętami skupiska są bardziej zwarte, a obie trajektorie wykazują większą zgodność.
Nie oznacza to jeszcze, że znamy źródło obserwowanych błędów. Rozbieżności mogą wynikać z odometrii, niedokładności lokalnych pomiarów kamerą lub ich przekształcenia do wspólnego układu współrzędnych. Na tym etapie możemy jedynie stwierdzić, że stopień niespójności zależy od warunków przejazdu.
Powtórna obserwacja nieruchomego landmarku dostarcza jednak informacji, której nie daje sama odometria. Ten sam punkt powinien mieć jedną pozycję globalną, niezależnie od przebytej drogi. Jeśli jego kolejne obserwacje nie pokrywają się, otrzymujemy informację o błędzie całego łańcucha pomiarowego.
W obecnym map fittingu nie wykorzystuję jeszcze tej informacji do poprawiania trajektorii. Pozostaje ona zamrożona, a wyznaczone centroidy i wartości RMS służą do oceny spójności obserwacji. Dopiero w kolejnym etapie będzie można sprawdzić, czy wspólna estymacja trajektorii pojazdu i położeń landmarków pozwoli zmniejszyć zaobserwowane rozbieżności.

### 10) co dalej
Dopiero kolejne kroki — korekta trajektorii, pełny SLAM i modelowanie poślizgu — mają sens. Ale to już osobna historia. Spróbuje to zrobić później. Na razie kamera zrobiła coś znacznie ważniejszego niż „zobaczenie pasa”. Stała się czujnikiem niespójności modelu ruchu. A to jest fundament każdego sensownego systemu lokalizacji.
