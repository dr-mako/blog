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
Po przejechaniu pętli mapa nie domyka się, a pas zaczyna “płynąć”. Obserwowana niespójność może wynikać z błędów odometrii, ale również z niedokładności pomiarów kamerą i przekształcenia BEV. Sam obraz złożonej mapy nie pozwala jeszcze rozdzielić tych przyczyn. Rys. 2 przedstawia kilka wybranych klatek obrazów z całego przejazdu złożonych w opisany powyżej sposób. Pasy ruchu rozjeżdzają się - szczególnie w okolicy ostrych zakrętów. Wynik pokazuje, że samo przeliczanie kolejnych obrazów BEV za pomocą odometrii nie gwarantuje uzyskania spójnej mapy. 

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
\mathbf{x}_k=(x_k,y_k,\varphi_k)
$$

Z detekcji markera otrzymujemy jego położenie na obrazie, a następnie, po przekształceniu do metrycznego BEV, współrzędne $(X_{\mathrm{BEV}},Y_{\mathrm{BEV}})$.
W rozpatrywanym przypadku zależność pomiędzy układem BEV a lokalnym układem pojazdu jest znana. Wynika ona z pomiaru geometrii stanowiska, w szczególności położenia planszy kalibracyjnej względem środka geometrycznego samochodu. Zależność tę uwzględniam już podczas wyznaczania przekształcenia obrazu do BEV. Otrzymane współrzędne metryczne są więc odniesione bezpośrednio do pojazdu, bez konieczności dodatkowego wyznaczania parametrów zewnętrznych kamery na etapie map fittingu.
Trzeba natomiast uwzględnić różnicę w orientacji osi obu układów. W układzie BEV oś $Y_{\mathrm{BEV}}$ wskazuje kierunek jazdy, a oś $X_{\mathrm{BEV}}$ kierunek poprzeczny. W lokalnym układzie odometrii oś $x$ jest skierowana do przodu, natomiast oś $y$ w lewo. 
Tą sytuację przedstawia rys:

<img src="{{ 'assets/images/CameraSensor/ekstrynsyka.png' | relative_url }}" alt="ekstrynsyka" style="width:100%; max-width:100%; height:auto;" />

W tym przypadku współrzędnych ma zatem postać:

$$
\mathbf{z}{k,j}^{\mathrm{local}}=
\begin{bmatrix}
Y{\mathrm{BEV}}\
-X_{\mathrm{BEV}}
\end{bmatrix}
$$

Tak otrzymany pomiar landmarku jest już wyrażony względem pojazdu. Aby przenieść go do układu globalnego, wystarczy obrócić go o kąt orientacji pojazdu i dodać jego aktualną pozycję:

$$
\begin{bmatrix}
x_k\
y_k
\end{bmatrix}
+
R(\varphi_k)\mathbf{z}_{k,j}^{\mathrm{local}}
$$

gdzie:

- $R(\varphi_k)$ – macierz obrotu wynikająca z orientacji pojazdu,

- $\mathbf{z}_{k,j}^{\mathrm{local}}$ – lokalny pomiar landmarku $j$ w chwili $k$,

- $\hat{\mathbf{p}}_{k,j}^{G}$ – globalna pozycja tego landmarku obliczona z pojedynczej obserwacji.

Ponieważ ten sam nieruchomy marker może być obserwowany w wielu klatkach, otrzymujemy zbiór jego globalnych pozycji. Gdyby odometria i pomiary kamery były idealne, wszystkie te punkty pokrywałyby się.

W praktyce tworzą skupisko. Jego środek wyznaczam jako średnią obserwacji:

$$
\bar{\mathbf{p}}j^G =
\frac{1}{N_j}
\sum{k=1}^{N_j}\hat{\mathbf{p}}_{k,j}^{G}
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

Następnie dla każdego identyfikatora markera obliczam centroid jego globalnych obserwacji oraz RMS. Trajektoria odometryczna i transformacja związana z geometrią kamery pozostają niezmienione. Otrzymany rozrzut jest więc miarą spójności obserwacji przy przyjętych parametrach całego układu pomiarowego.


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

*W tabeli uwzględniono markery z co najmniej pięcioma obserwacjami.*


Rozrzut nie jest jednakowy. Największą spójność wykazują obserwacje markera ID 1, dla którego RMS wynosi 6,3 mm. Z kolei dla markera ID 9 otrzymano 52,5 mm. Średni RMS, obliczony dla dziesięciu uwzględnionych landmarków, wynosi 27,25 mm.
Wyznaczone centroidy są reprezentatywnymi pozycjami landmarków przy przyjętej trajektorii odometrycznej. Nie oznacza to jednak, że odpowiadają ich rzeczywistemu położeniu. Nawet niewielki RMS może występować wtedy, gdy wszystkie obserwacje danego markera są przesunięte w podobny sposób. Dlatego spójność wewnętrzną mapy trzeba odróżnić od jej zgodności z rzeczywistą geometrią stanowiska.

### 8) Czy geometria trasy wpływa na spójność mapy?

W pierwszym eksperymencie samochód poruszał się po stosunkowo krótkiej trasie z ostrymi zakrętami. Rozrzut globalnych obserwacji landmarków był wyraźny, a średni RMS wyniósł 27,25 mm. Największe różnice pojawiały się w obszarach związanych z intensywnymi manewrami.
Postanowiłem sprawdzić, jak zmienią się wyniki, gdy samochód będzie poruszał się po innej trasie. W drugim eksperymencie wykorzystałem dłuższy tor o łagodniejszych zakrętach. Sposób przetwarzania danych pozostał taki sam: obserwacje markerów przeliczam przez metryczne BEV do układu pojazdu, a następnie przenoszę do układu globalnego na podstawie odometrii. Trajektoria nadal pozostaje zamrożona. Na rys. 7 przedstawiono trajektorie dwóch przejazdów oraz globalne obserwacje landmarków dla drugiego eksperymentu.

<img src="{{ 'assets/images/CameraSensor/Przejazd 2 przed.png' | relative_url }}" alt="Przejazd 2 przed" style="width:100%; max-width:100%; height:auto;" />

Różnica jest wyraźna. Obserwacje tych samych markerów tworzą znacznie bardziej zwarte skupiska. W drugim eksperymencie wartości RMS dla większości landmarków mieszczą się w granicach kilkunastu milimetrów, a największa z przedstawionych w zestawieniu wynosi 15,3 mm.
To interesujący wynik. Pomimo zwiększenia długości trasy uzyskałem mniejszy rozrzut obserwacji. Sugeruje to, że sama długość przejazdu nie decyduje o spójności mapy. Znaczenie mogą mieć również geometria toru i warunki, w jakich porusza się samochód.
Nie można jednak na podstawie tych dwóch eksperymentów jednoznacznie wskazać przyczyny zaobserwowanej różnicy. Na wyniki mogły wpływać warunki ruchu, dokładność odometrii oraz jakość lokalnych pomiarów z kamery. Aby rozdzielić wpływ poszczególnych czynników, potrzebne byłyby kolejne eksperymenty.
Na tym etapie istotne jest samo porównanie: przy zastosowaniu tej samej metody tworzenia BEV i map fittingu otrzymałem wyraźnie różne wartości rozrzutu obserwacji. Wyniki obu eksperymentów mogą więc posłużyć jako punkt odniesienia w dalszych pracach nad korekcją trajektorii.
Pozostaje pytanie, czy wykorzystanie powtarzających się obserwacji nieruchomych landmarków pozwoli zmniejszyć te rozbieżności. Do tego wrócę przy implementacji SLAM-u.

### 9) Jak druga pętla ujawnia dryf odometrii?
Dotychczas porównywałem rozrzut obserwacji landmarków dla dwóch różnych tras. Warto jednak przyjrzeć się jeszcze jednej sytuacji: co dzieje się, gdy samochód ponownie przejeżdża tę samą trasę i obserwuje te same nieruchome markery? 
W obu eksperymentach samochód wykonywał dwa przejazdy po swoim torze. Na rys. 8 zestawiono trajektorie odometryczne oraz globalne obserwacje landmarków dla obu eksperymentów. Linie ciągłe i przerywane odpowiadają kolejnym przejazdom, natomiast kolory punktów oznaczają identyfikatory markerów.

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

Markery są nieruchome, więc ich rzeczywiste położenia nie zmieniają się pomiędzy przejazdami. Gdyby odometria i pomiary z kamery były idealne, obserwacje tego samego markera trafiałyby w jedno miejsce na mapie, niezależnie od tego, w której pętli zostały zarejestrowane.
Na wykresach widać jednak różnice. W pierwszym eksperymencie, na trasie z ostrymi zakrętami, obserwacje tych samych landmarków są wyraźnie rozdzielone pomiędzy przejazdami. W drugim eksperymencie, na dłuższej trasie z łagodniejszymi zakrętami, różnice są znacznie mniejsze.
Obserwujemy zatem dwa zjawiska. W kolejnych klatkach tego samego przejazdu pozycje markera tworzą skupisko o określonym rozrzucie. Po ponownym przejechaniu trasy skupisko może pojawić się w innym miejscu mapy, mimo że marker nie zmienił swojego rzeczywistego położenia.
To rozróżnienie jest istotne. Niewielki RMS obserwacji z jednego przejazdu nie gwarantuje, że po ponownym przejechaniu trasy otrzymamy tę samą globalną pozycję landmarku.
Na tym etapie nie potrafię jeszcze jednoznacznie rozdzielić błędów odometrii od niedokładności pomiarów kamerą. Mogę natomiast sprawdzić, czy ponowne obserwacje nieruchomych punktów są zgodne z mapą utworzoną podczas pierwszego przejazdu.
Obecny map fitting nie wykorzystuje tych rozbieżności do korekcji trajektorii. Przyjmuje ją za ustaloną, a pozycje landmarków wyznacza na podstawie globalnych obserwacji.
W kolejnym etapie zamierzam wykorzystać powtarzające się obserwacje do wspólnej estymacji trajektorii pojazdu i położeń landmarków. Pozwoli to sprawdzić, czy można zmniejszyć niespójność mapy bez zakładania, że odometria jest idealna.

### 10) co dalej
Map fitting pozwolił mi sprawdzić, jak spójne są obserwacje nieruchomych landmarków przy przyjętej trajektorii odometrycznej. Porównanie dwóch tras pokazało wyraźne różnice w rozrzucie obserwacji, natomiast ponowne przejazdy ujawniły rozbieżności pomiędzy globalnymi pozycjami tych samych markerów. 
Na razie nie rozstrzygam, jaka część tych błędów wynika z odometrii, a jaka z pomiarów kamerą. To będzie przedmiotem dalszych badań.
Kolejnym krokiem jest SLAM, w którym trajektoria pojazdu przestanie być traktowana jako niezmienna. Będzie można wtedy sprawdzić, czy uwzględnienie obserwacji landmarków pozwoli poprawić lokalizację i uzyskać bardziej spójną mapę.
