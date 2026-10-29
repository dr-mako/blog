---
layout: post
title: "Korekta obrazu (bird’s eye view)"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Po co nam to potrzebne?<!--more-->


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


### 1) Koncepcja
W projekcie korzystam z kamery typu rybie oko $170^\circ$. Powód jest prosty: kamera jest zamontowana nisko, a ja chcę widzieć możliwie dużo „podłogi” i toru jazdy tuż przed pojazdem. Przy klasycznym obiektywie z takiej pozycji szybko tracę pole widzenia, natomiast rybie oko daje szeroki kadr i dużo informacji o przebiegu pasa.
Cena za ten komfort jest jednak wysoka: obraz z rybiego oka jest silnie zniekształcony. Linie, które w rzeczywistości są proste, w obrazie stają się krzywe, a skala obiektów mocno zależy od położenia w kadrze. Ten efekt najłatwiej zauważyć na szachownicy kalibracyjnej — rys. poniżej pokazuje ustawienie pojazdu podczas wykonywania zdjęcia oraz przykładowy obraz z kamery (rys. 1).

<img src="{{ 'assets/images/Widok_z_lotu/F1.png' | relative_url }}" alt="F1" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 1. Ustawienie pojazdu podczas wykonywania zdjęcia szachownicy oraz przykładowy obraz z kamery.</i></p>

Dla człowieka „wygięta” szachownica nie stanowi problemu — mózg koryguje to automatycznie. Dla algorytmu sterowania, który ma wykrywać krawędzie pasa i utrzymywać pojazd w jego środku, jest to już realna przeszkoda. To samo miejsce na torze może wyglądać inaczej w kolejnych klatkach, a proste operacje geometryczne przestają działać stabilnie.
Najlepsze własności geometryczne ma sytuacja, w której obserwujemy podłogę prostopadle z góry — „z lotu ptaka”. Taki obraz ma w przybliżeniu stałą skalę, a linie na podłodze pozostają liniami. Dokładnie tego chcę. Dlatego wprowadzam korektę obrazów do widoku z lotu ptaka (bird’s eye view, BEV). Zakładam przy tym, że interesuje mnie przede wszystkim płaszczyzna drogi (mata/tor), a wszystko, co jest „pionowe”, traktuję jako efekt uboczny. BEV nie jest rekonstrukcją 3D — to świadome uproszczenie: buduję przekształcenie 2D, które na podłodze daje obraz o bardziej stałej geometrii, wygodny do sterowania, składania mapy z wielu klatek oraz ewentualnego uczenia sieci.
Przekształcenie (fisheye $\rightarrow$ bird’s eye / usunięcie dystorsji) musi się jednak „z czegoś wziąć” — potrzebuję informacji, jak piksele obrazu odpowiadają punktom na podłodze. Są dwa sensowne sposoby, żeby tę zależność wyznaczyć.

### 2) Dwa sposoby wyznaczania BEV
Rozpatruję dwa podejścia do przekształcania obrazu.
Pierwsze wykorzystuje bezpośrednio punkty pomiarowe i interpolację. Drugie opiera się na modelu kamery fisheye oraz homografii.

## Interpolacja punktów pomiarowych

W tym podejściu nie muszę najpierw wyznaczać parametrów optycznych kamery. Wykorzystuję zbiór odpowiadających sobie punktów obrazu i płaszczyzny drogi.
Jeśli znam współrzędne punktu na podłodze oraz jego położenie na obrazie, mogę na podstawie wielu takich par zbudować funkcję odwzorowania.
W programie wykorzystuję interpolację liniową w dwóch wymiarach. Dla zadanych współrzędnych metrycznych wyznaczam odpowiadające im współrzędne pikselowe w obrazie źródłowym.
Zaletą jest bezpośrednie wykorzystanie danych pomiarowych. Ograniczenie pojawia się poza obszarem pokrytym punktami. Tam interpolacja nie daje już wyniku, a prosta ekstrapolacja mogłaby prowadzić do dużych błędów.

## Model kamery i homografia

W drugim podejściu najpierw wyznaczam parametry kamery fisheye, a następnie koryguję dystorsję obrazu. 
Dopiero dla tak skorygowanego obrazu wyznaczam homografię, czyli transformację projektową pomiędzy płaszczyzną obrazu a płaszczyzną planszy kalibracyjnej. 
Homografia opisuje jedną zależność obowiązującą na całej rozpatrywanej płaszczyźnie. Umożliwia też obliczanie odwzorowania poza obszarem bezpośrednio pokrytym punktami kalibracyjnymi, choć dokładność takiego przeliczenia zależy od jakości wyznaczonej transformacji. 
Oba podejścia prowadzą do obrazu BEV, lecz w inny sposób opisują geometrię. Ich porównanie pozwala sprawdzić, jak wynik zależy od przyjętej metody.

### 3) Plansza kalibracyjna $ChArUco$
Kluczowym elementem całego procesu jest sposób pozyskania danych. Zamiast ręcznego wskazywania punktów wykorzystuję planszę typu ChArUco, która łączy szachownicę z markerami ArUco.
Planszę wykorzystaną w projekcie przedstawiono na rys. 2.

<img src="{{ 'assets/images/Widok_z_lotu/charuco_840x420.png' | relative_url }}" alt="charuco_840x420" style="width:45%; max-width:100%; height:auto;" />
<p><i>Rys. 2. Plansza kalibracyjna ChArUco wykorzystująca słownik markerów DICT_6X6_250.</i></p>

Każdy marker posiada unikalny identyfikator, dzięki czemu możliwe jest jednoznaczne przypisanie wykrytych punktów do ich pozycji na planszy. W praktyce oznacza to, że nawet przy częściowym widoku planszy możliwe jest uzyskanie poprawnych korespondencji.
Po wykryciu markera, program zapisuje odpowiadające sobie współrzędne:

- $((u,v))$ – położenie narożnika w obrazie, wyrażone w pikselach,

- $((X_B,Y_B))$ – położenie tego narożnika na planszy, wyrażone w milimetrach.

Dzięki temu otrzymuję zbiór korespondencji pomiędzy obrazem a rzeczywistą geometrią stanowiska. 
Efektem działania algorytmu detekcji jest zbiór danych, w którym każdemu punktowi obrazu odpowiada punkt na planszy. Etapy procesu detekcji markerów przedstawiam na rys. 3: wykonanie zdjęcia, obraz z kamery, wynik detekcji markerów i szczegóły oznaczenia identyfikatorów narożników. 

<img src="{{ 'assets/images/Widok_z_lotu/Uklad.png' | relative_url }}" alt="Uklad" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 3. Przykład detekcji markerów ChArUco: obraz wejściowy, wykryte markery oraz szczegóły (ID, narożniki, ramki).</i></p>

Powstaje w ten sposób gęsta, metryczna tabela odwzorowania przestrzeni. Ten sam zbiór punktów może służyć do budowy interpolacji, wyznaczania homografii i kalibracji modelu kamery.

### 4) Model kamery fisheye
Niezależnie od sposobu pozyskiwania punktów korespondencji pomiędzy obrazem a podłogą — czy jest to metoda ręczna oparta na szachownicy, czy automatyczna detekcja z wykorzystaniem planszy ChArUco — możliwe jest zbudowanie modelu kamery. Biblioteka OpenCV udostępnia w tym celu gotowe narzędzia, które na podstawie wykrytych narożników estymują parametry optyczne układu.
Warunkiem działania tej procedury jest wykonanie serii zdjęć planszy kalibracyjnej przy zachowaniu stałej geometrii układu, w szczególności niezmiennego położenia kamery względem podłogi. Przykładowy zestaw takich obrazów przedstawiono na rys. 4.

<img src="{{ 'assets/images/Widok_z_lotu/mozaika_frame.png' | relative_url }}" alt="mozaika_frame" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 4. Zestaw obrazów wykorzystanych do kalibracji kamery typu rybie oko.</i></p>

W wyniku kalibracji otrzymywany jest model kamery, który można interpretować jako zestaw parametrów opisujących transformację pomiędzy przestrzenią trójwymiarową a obrazem. Zapisuje się go następująco:

macierz parametrów wewnętrznych (K), zawierająca ogniskowe i współrzędne punktu głównego,

współczynniki dystorsji (D), opisujące zniekształcenia obiektywu.

Podczas kalibracji wyznaczane są również wektory obrotu i translacji dla poszczególnych obserwacji planszy. Parametry te opisują jej położenie względem kamery i są potrzebne w procesie kalibracji oraz do analizy błędu reprojekcji. 
Nie wykorzystuję ich jednak do ustalania położenia układu współrzędnych pojazdu w końcowej transformacji BEV. 
Po wyznaczeniu parametrów $(K)$ i $(D)$ mogę skorygować obraz fisheye, otrzymując obraz o geometrii perspektywicznej. To punkt wyjścia do wyznaczenia homografii.

### 5) Od obrazu do układu planszy
Po korekcji dystorsji znam współrzędne wykrytych narożników w obrazie skorygowanym. Znam też ich rzeczywiste położenia na planszy. 
Na tej podstawie wyznaczam macierz homografii $(H_{\mathrm{board}})$. 
Dla punktu obrazu po korekcji dystorsji zapisuję zależność:

$$
H_{\mathrm{board}}
\begin{bmatrix}
u_{\mathrm{ud}}\v_{\mathrm{ud}}\1
\end{bmatrix}
]
$$

gdzie $(\lambda)$ jest współczynnikiem skali współrzędnych jednorodnych.
Po normalizacji otrzymuję współrzędne punktu na płaszczyźnie planszy. To ważny etap. Punkt, który na obrazie był opisany tylko przez dwa numery pikseli, otrzymuje teraz położenie geometryczne wyrażone w milimetrach. Wciąż jednak są to współrzędne związane z planszą, a nie bezpośrednio z pojazdem.

### 6) Od układu planszy do układu pojazdu
W projekcie punktem odniesienia jest środek geometryczny samochodu (O). Z nim związany jest lokalny układ współrzędnych używany również w odometrii.
Oś $(X_{\mathrm{local}})$ opisuje kierunek poprzeczny, natomiast oś $(Y_{\mathrm{local}})$ jest skierowana zgodnie z kierunkiem jazdy.
Aby przeliczyć współrzędne planszy do tego układu, wykorzystuję znane wymiary stanowiska.
Szerokość planszy wynosi 810 mm, więc jej środek poprzeczny znajduje się w odległości 405 mm od krawędzi. 
Znany jest również rozstaw osi samochodu $(L=260)$ mm oraz promień koła $(R=37{,}25)$ mm.

Z przyjętego ustawienia planszy względem pojazdu wynika przesunięcie:

$$
[
\Delta L=\frac{L}{2}+R=167{,}25\ \mathrm{mm}
]
$$

Przeliczenie współrzędnych można więc zapisać jako:

$$
[
X_{\mathrm{local}}=405-X_B
]
$$
$$
[
Y_{\mathrm{local}}=167{,}25+Y_B
]
$$

lub w postaci macierzowej:

$$
[
T_{\mathrm{board}\to\mathrm{vehicle}}=
\begin{bmatrix}
-1&0&405\
0&1&167{,}25\
0&0&1
\end{bmatrix}
]
$$

Współczynnik (-1) odpowiada za przeciwny zwrot osi poprzecznej planszy względem przyjętej osi lokalnej pojazdu. 
Ta transformacja nie jest wyznaczana z parametrów zewnętrznych kamery. Wynika z fizycznego pomiaru i ustawienia planszy względem samochodu. 
Nie należy jej zatem utożsamiać z parametrami $(R,t)$, które OpenCV estymuje podczas kalibracji kamery.
Łącząc homografię z transformacją układów współrzędnych, otrzymuję:

$$
[
H=T_{\mathrm{board}\to\mathrm{vehicle}},
H_{\mathrm{board}}
]
$$

Dzięki temu punkt obrazu po korekcji dystorsji mogę przeliczyć bezpośrednio na współrzędne metryczne w lokalnym układzie pojazdu.

### 7) Generowanie obrazu BEV
Znając transformację obrazu do płaszczyzny pojazdu, mogę utworzyć wynikowy obraz BEV.
Przyjmuję regularną siatkę o rozdzielczości 2 mm na piksel. Każdy punkt tej siatki odpowiada określonemu położeniu na podłodze względem samochodu.
Aby wypełnić obraz, stosuję odwzorowanie odwrotne. Dla każdego piksela obrazu BEV obliczam, z którego miejsca obrazu źródłowego należy pobrać wartość.
Wynikiem są dwie macierze:

- Xmap – współrzędne poziome punktów obrazu źródłowego,

- Ymap – współrzędne pionowe punktów obrazu źródłowego.

Następnie funkcja cv2.remap tworzy obraz wynikowy na podstawie tych map.
W metodzie interpolacyjnej mapy powstają bezpośrednio z interpolacji punktów pomiarowych.
W podejściu opartym na modelu kamery mapy wynikają z korekcji dystorsji i odwrotnej homografii. Możliwe jest także rozszerzenie obszaru BEV poza granice planszy kalibracyjnej, z zastrzeżeniem większej niepewności takiego odwzorowania.
Warto rozróżnić metodę wyznaczenia transformacji od sposobu jej późniejszego zastosowania. Macierze Xmap i Ymap są reprezentacją gotowego odwzorowania. Same nie poprawiają dokładności geometrycznej ani nie usuwają błędów kalibracji.

### 8) Porównanie wyników między interpolacją, a modelem kamery
Obie metody wykorzystują tę samą rzeczywistą geometrię stanowiska, ale inaczej wyznaczają zależność pomiędzy obrazem a podłogą.
Interpolacja opiera się na lokalnych korespondencjach. W obrębie obszaru pomiarowego nie narzuca globalnego modelu projekcyjnego. Poza tym obszarem nie wyznacza jednak poprawnych wartości bez zastosowania dodatkowej metody. 
Homografia wykorzystuje jedno globalne przekształcenie płaszczyzny. Wymaga wcześniejszego usunięcia dystorsji fisheye, a jej dokładność zależy zarówno od kalibracji kamery, jak i od poprawności wyznaczenia macierzy $(H)$.
Porównanie obrazów pozwala ocenić deformacje, zachowanie geometrii siatki i zakres powierzchni, który można odtworzyć.
Nie traktuję jednak samego wyglądu obrazu jako dowodu dokładności metrycznej. Do jej oceny potrzebne jest porównanie przeliczonych współrzędnych z punktami o znanym rzeczywistym położeniu.
Z punktu widzenia dalszej lokalizacji samochodu najważniejsze jest to, czy punkt na podłodze otrzymuje poprawne współrzędne względem pojazdu.

<img src="{{ 'assets/images/Widok_z_lotu/BEV.png' | relative_url }}" alt="BEV" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 5. Porównanie obrazu BEV uzyskanego metodą interpolacji (po prawej) oraz homografii (po lewej).</i></p>

Różnice pomiędzy podejściem lokalnym i globalnym są dobrze widoczne na rys. 5. Interpolacja zachowuje wysoką dokładność w obszarach pokrytych punktami pomiarowymi, natomiast homografia zapewnia ciągłość obrazu kosztem deformacji geometrycznych, szczególnie na obrzeżach.

<img src="{{ 'assets/images/Widok_z_lotu/Difference.png' | relative_url }}" alt="DifferenceV" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 6. Analiza różnic pomiędzy metodami: maska obszarów dodatkowych, mapa różnic oraz próba fuzji wyników.</i></p>

Próba bezpośredniego połączenia obu metod prowadzi jedynie do poprawy wizualnej, ale nie tworzy jednego spójnego modelu odwzorowania. Interpolacja i homografia opisują przestrzeń w różny sposób — odpowiednio lokalny i globalny — dlatego ich wyniki nie są bezpośrednio kompatybilne (rys. 6).

### 9) Wybór rozwiązania: mapy $Xmap$ i $Ymap$
Zamiast wybierać jedną z metod w czystej postaci, stosuję podejście pośrednie oparte na jawnej mapie przekształcenia. Buduję dwie macierze: $Xmap$ oraz $Ymap$, które dla każdego punktu obrazu wynikowego wskazują odpowiadający mu punkt w obrazie źródłowym. 
W praktyce oznacza to, że całe przekształcenie sprowadza się do operacji remapowania obrazu. Rozwiązanie to nie wymaga ani idealnego modelu kamery, ani globalnego uproszczenia geometrii. Jednocześnie zachowuje wysoką dokładność lokalną, ponieważ opiera się bezpośrednio na danych pomiarowych, oraz zapewnia pełne pokrycie obrazu dzięki interpolacyjnemu wypełnieniu przestrzeni. 
Można więc powiedzieć, że jest to kompromis pomiędzy podejściem lokalnym i globalnym — wykorzystuje rzeczywiste dane, ale prowadzi do jednego, spójnego odwzorowania.
Efektem jest stabilny, metryczny obraz podłogi, który nadaje się zarówno do sterowania pojazdem, jak i do budowy mapy z wielu klatek.

### 10) Ograniczenia 
Obie metody zakładają, że obserwowane punkty leżą na płaszczyźnie drogi. Elementy znajdujące się ponad nią zostaną w obrazie BEV przekształcone niezgodnie z ich rzeczywistą geometrią.
Istotna jest również stałość położenia kamery względem samochodu. Zmiana wysokości montażu, kąta nachylenia lub położenia kamery powoduje zmianę zależności między obrazem a podłogą.
W metodzie interpolacyjnej ograniczeniem jest zakres dostępnych danych pomiarowych. W metodzie opartej na homografii należy dodatkowo uwzględnić błędy kalibracji oraz niepewność odwzorowania poza obszarem, w którym znajdują się punkty użyte do jej wyznaczenia.
Dokładność końcowego pomiaru zależy też od poprawności przyjętej geometrii stanowiska. Błąd w wymiarach lub ustawieniu planszy względem samochodu zostanie przeniesiony na współrzędne lokalne.

### 11) Co dalej
Celem całego przekształcenia jest uzyskanie z obrazu kamery pomiarów geometrycznych, które można bezpośrednio powiązać z ruchem pojazdu.
Po detekcji punktu na płaszczyźnie drogi mogę przeliczyć jego współrzędne pikselowe na współrzędne lokalne $((X_{\mathrm{local}}$, $Y_{\mathrm{local}}))$.
Jeżeli znam również położenie i orientację samochodu z odometrii, mogę przenieść taki punkt do wspólnego układu mapy.
W ten sposób obserwacje tych samych nieruchomych elementów toru z kolejnych klatek pozwolą sprawdzać spójność odometrii. Rozbieżności ich położenia mogą wskazywać na błędy modelu ruchu, poślizg lub niepewność pomiaru.
Kamera przestaje więc być jedynie źródłem obrazu do wykrywania pasa. Dostarcza również informacji o położeniu punktów względem samochodu, które można wykorzystać do dalszej budowy mapy i korekcji lokalizacji.
