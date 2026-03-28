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
Cena za ten komfort jest jednak wysoka: obraz z rybiego oka jest silnie zniekształcony. Linie, które w rzeczywistości są proste, w obrazie stają się krzywe, a skala obiektów mocno zależy od położenia w kadrze. Ten efekt najłatwiej zauważyć na szachownicy kalibracyjnej — rys. poniżej pokazuje ustawienie pojazdu podczas wykonywania zdjęcia oraz przykładowy obraz z kamery.

<img src="{{ 'assets/images/Widok_z_lotu/F1.png' | relative_url }}" alt="F1" style="width:75%; max-width:100%; height:auto;" />

Dla człowieka „wygięta” szachownica nie stanowi problemu — mózg koryguje to automatycznie. Dla algorytmu sterowania, który ma wykrywać krawędzie pasa i utrzymywać pojazd w jego środku, jest to już realna przeszkoda. To samo miejsce na torze może wyglądać inaczej w kolejnych klatkach, a proste operacje geometryczne przestają działać stabilnie.
Najlepsze własności geometryczne ma sytuacja, w której obserwujemy podłogę prostopadle z góry — „z lotu ptaka”. Taki obraz ma w przybliżeniu stałą skalę, a linie na podłodze pozostają liniami. Dokładnie tego chcę. Dlatego wprowadzam korektę obrazów do widoku z lotu ptaka (bird’s eye view, BEV). Zakładam przy tym, że interesuje mnie przede wszystkim płaszczyzna drogi (mata/tor), a wszystko, co jest „pionowe”, traktuję jako efekt uboczny. BEV nie jest rekonstrukcją 3D — to świadome uproszczenie: buduję przekształcenie 2D, które na podłodze daje obraz o bardziej stałej geometrii, wygodny do sterowania, składania mapy z wielu klatek oraz ewentualnego uczenia sieci.
Przekształcenie (fisheye $\rightarrow$ bird’s eye / usunięcie dystorsji) musi się jednak „z czegoś wziąć” — potrzebuję informacji, jak piksele obrazu odpowiadają punktom na podłodze. Są dwa sensowne sposoby, żeby tę zależność wyznaczyć.

### 2) Metody przekształcania
Aby uzyskać widok z lotu ptaka, konieczne jest określenie zależności pomiędzy pikselami obrazu a punktami na podłodze. Problem ten można rozwiązać na kilka sposobów, które różnią się przede wszystkim tym, czy opisują odwzorowanie lokalnie, czy globalnie.
Najbardziej bezpośrednim podejściem jest interpolacja oparta na punktach pomiarowych. W tym przypadku nie buduje się modelu kamery — zamiast tego wykorzystuje się zbiór znanych korespondencji pomiędzy obrazem a przestrzenią i konstruuje odwzorowanie lokalne. Metoda ta pozwala uzyskać bardzo wysoką dokładność tam, gdzie dostępne są dane, jednak jej działanie ogranicza się do obszaru pokrytego punktami. Na brzegach pojawiają się braki danych i niestabilność.
Drugim podejściem jest wykorzystanie homografii, czyli globalnego przekształcenia projektowego. Zakłada ono, że scena jest płaska, a obraz został wcześniej skorygowany z dystorsji. Homografia zapewnia spójność całego obrazu, ale upraszcza rzeczywistą geometrię — szczególnie w przypadku obiektywu typu rybie oko, gdzie deformacje są znaczne.
Trzecia możliwość polega na wykorzystaniu modelu kamery. W tym przypadku estymowane są parametry optyczne, a następnie używane do rzutowania promieni na płaszczyznę. Podejście to jest fizycznie poprawne, ale w praktyce okazuje się wrażliwe na błędy kalibracji, zwłaszcza w obszarach peryferyjnych.
W efekcie żadna z metod nie jest idealna. Interpolacja daje wysoką dokładność lokalną, homografia zapewnia stabilność globalną, a model kamery oferuje spójny opis optyki, ale kosztem złożoności i wrażliwości na błędy. Wybór rozwiązania staje się więc kompromisem pomiędzy tymi cechami.


### 3) Plansza kalibracyjna $ChArUco$
Kluczowym elementem całego procesu jest sposób pozyskania danych. Zamiast ręcznego wskazywania punktów wykorzystuję planszę typu ChArUco, która łączy szachownicę z markerami ArUco.
Planszę wykorzystaną w projekcie przedstawiono na rys. 1.

<img src="{{ 'assets/images/Widok_z_lotu/charuco_840x420.png' | relative_url }}" alt="charuco_840x420" style="width:15%; max-width:100%; height:auto;" />
<p><i>Rys. 1. Plansza kalibracyjna ChArUco wykorzystująca słownik markerów DICT_6X6_250.</i></p>

Każdy marker posiada unikalny identyfikator, dzięki czemu możliwe jest jednoznaczne przypisanie wykrytych punktów do ich pozycji na planszy. W praktyce oznacza to, że nawet przy częściowym widoku planszy możliwe jest uzyskanie poprawnych korespondencji.
Efektem działania algorytmu detekcji jest zbiór danych, w którym każdemu punktowi obrazu odpowiada punkt na planszy. Etapy procesu detekcji markerów przedstawiam na rys. 2: wykonanie zdjęcia, obraz z kamery, wynik detekcji markerów i szczegóły oznaczenia identyfikatorów narożników. 

<img src="{{ 'assets/images/Widok_z_lotu/Uklad.png' | relative_url }}" alt="Uklad" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 2. Przykład detekcji markerów ChArUco: obraz wejściowy, wykryte markery oraz szczegóły (ID, narożniki, ramki).</i></p>

Powstaje w ten sposób gęsta, metryczna tabela odwzorowania przestrzeni. W porównaniu do ręcznego klikania pozwala to uzyskać znacznie większą liczbę punktów, a jednocześnie eliminuje błędy wynikające z niejednoznaczności.

### 4) Model kamery
Niezależnie od sposobu pozyskiwania punktów korespondencji pomiędzy obrazem a podłogą — czy jest to metoda ręczna oparta na szachownicy, czy automatyczna detekcja z wykorzystaniem planszy ChArUco — możliwe jest zbudowanie modelu kamery. Biblioteka OpenCV udostępnia w tym celu gotowe narzędzia, które na podstawie wykrytych narożników estymują parametry optyczne układu.
Warunkiem działania tej procedury jest wykonanie serii zdjęć planszy kalibracyjnej przy zachowaniu stałej geometrii układu, w szczególności niezmiennego położenia kamery względem podłogi. Przykładowy zestaw takich obrazów przedstawiono na rys. 3.

<img src="{{ 'assets/images/Widok_z_lotu/mozaika_frame.png' | relative_url }}" alt="mozaika_frame" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 3. Zestaw obrazów wykorzystanych do kalibracji kamery typu rybie oko.</i></p>

W wyniku kalibracji otrzymywany jest model kamery, który można interpretować jako zestaw parametrów opisujących transformację pomiędzy przestrzenią trójwymiarową a obrazem. W jego skład wchodzą między innymi ogniskowa, położenie środka optycznego oraz współczynniki dystorsji, które modelują deformacje charakterystyczne dla obiektywów szerokokątnych. Model ten pozwala na „odwrócenie” zniekształceń i rzutowanie promieni kamery na płaszczyznę podłogi, co teoretycznie umożliwia uzyskanie widoku z lotu ptaka w sposób fizycznie poprawny. 
W praktyce pojawia się jednak istotny problem. Estymacja parametrów odbywa się na podstawie skończonej liczby punktów i zawsze obarczona jest błędem. Błąd ten jest niewielki w centralnej części obrazu, ale rośnie w jego obszarach peryferyjnych, gdzie dystorsja obiektywu jest największa. W rezultacie model kamery, choć spójny geometrycznie, nie odwzorowuje idealnie relacji pomiędzy obrazem a płaszczyzną podłogi. To ograniczenie ma bezpośredni wpływ na jakość obrazu BEV i stanowi jeden z powodów, dla których warto porównać to podejście z innymi metodami. Oznacza to, że sam model kamery nie jest wystarczający do uzyskania stabilnego i metrycznie poprawnego obrazu BEV.

### 5) Porównanie podejść
Dysponując zarówno danymi pomiarowymi, jak i modelem kamery, można przeanalizować zachowanie poszczególnych metod przekształcania obrazu do widoku z lotu ptaka.
Interpolacja oparta na punktach pomiarowych zapewnia bardzo wysoką dokładność w obszarach, gdzie dane są dostępne. Jednocześnie jednak nie radzi sobie poza nimi — pojawiają się braki danych i artefakty. Metoda ta ma więc charakter lokalny.
Homografia reprezentuje podejście przeciwne. Zapewnia spójność całego obrazu dzięki globalnemu modelowi przekształcenia, ale kosztem dokładności. Błąd ma charakter systematyczny i rośnie wraz z odległością od obszaru najlepszego dopasowania.
Model kamery stanowi próbę fizycznie poprawnego opisu układu, jednak w praktyce jego dokładność ograniczona jest przez błędy kalibracji, szczególnie w obszarach oddalonych od środka obrazu.
W efekcie mamy do czynienia z dwoma przeciwstawnymi podejściami: lokalnym i globalnym. Jedno zapewnia wysoką dokładność, drugie stabilność. Żadne z nich nie daje obu jednocześnie.

<img src="{{ 'assets/images/Widok_z_lotu/BEV.png' | relative_url }}" alt="BEV" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 4. Porównanie obrazu BEV uzyskanego metodą interpolacji (po prawej) oraz homografii (po lewej).</i></p>

Różnice pomiędzy podejściem lokalnym i globalnym są dobrze widoczne na rys. 4. Interpolacja zachowuje wysoką dokładność w obszarach pokrytych punktami pomiarowymi, natomiast homografia zapewnia ciągłość obrazu kosztem deformacji geometrycznych, szczególnie na obrzeżach.

<img src="{{ 'assets/images/Widok_z_lotu/Difference.png' | relative_url }}" alt="DifferenceV" style="width:75%; max-width:100%; height:auto;" />
<p><i>Rys. 5. Analiza różnic pomiędzy metodami: maska obszarów dodatkowych, mapa różnic oraz próba fuzji wyników.</i></p>

Próba bezpośredniego połączenia obu metod prowadzi jedynie do poprawy wizualnej, ale nie tworzy jednego spójnego modelu odwzorowania. Interpolacja i homografia opisują przestrzeń w różny sposób — odpowiednio lokalny i globalny — dlatego ich wyniki nie są bezpośrednio kompatybilne (rys. 5).

### 6) Wybór rozwiązania: mapy $Xmap$ i $Ymap$
Zamiast wybierać jedną z metod w czystej postaci, stosuję podejście pośrednie oparte na jawnej mapie przekształcenia. Buduję dwie macierze: $Xmap$ oraz $Ymap$, które dla każdego punktu obrazu wynikowego wskazują odpowiadający mu punkt w obrazie źródłowym. 
W praktyce oznacza to, że całe przekształcenie sprowadza się do operacji remapowania obrazu. Rozwiązanie to nie wymaga ani idealnego modelu kamery, ani globalnego uproszczenia geometrii. Jednocześnie zachowuje wysoką dokładność lokalną, ponieważ opiera się bezpośrednio na danych pomiarowych, oraz zapewnia pełne pokrycie obrazu dzięki interpolacyjnemu wypełnieniu przestrzeni. 
Można więc powiedzieć, że jest to kompromis pomiędzy podejściem lokalnym i globalnym — wykorzystuje rzeczywiste dane, ale prowadzi do jednego, spójnego odwzorowania.
Efektem jest stabilny, metryczny obraz podłogi, który nadaje się zarówno do sterowania pojazdem, jak i do budowy mapy z wielu klatek.

### 7) Ograniczenia 
Przyjęte podejście ma jednak swoje ograniczenia. Przede wszystkim działa poprawnie tylko dla konkretnej konfiguracji kamery — jej wysokości, kąta nachylenia oraz pola widzenia. Zmiana któregokolwiek z tych parametrów wymaga ponownego wyznaczenia map. 
Drugim ograniczeniem jest założenie płaskości sceny. Obiekty wystające ponad powierzchnię podłogi są w widoku z lotu ptaka zniekształcone, co jest bezpośrednią konsekwencją przyjętego modelu. 
W praktyce oznacza to, że odwzorowanie jest poprawne tylko dla płaszczyzny drogi, natomiast elementy przestrzenne nie są reprezentowane w sposób geometrycznie zgodny z rzeczywistością.

### 7) Co dalej
W kolejnym kroku obraz w układzie drogi — w szczególności jego wariant krawędziowy — zostanie wykorzystany do składania wielu klatek w jedną spójną mapę przejazdu. Kluczową rolę odgrywa tutaj odometria pojazdu, która dostarcza przybliżonej informacji o przesunięciach i obrotach, natomiast korekcja wizyjna pozwala kompensować błędy wynikające z poślizgów oraz niedokładności modelu ruchu. 
W praktyce oznacza to połączenie dwóch źródeł informacji: predykcji ruchu wynikającej z odometrii oraz dopasowania obrazu, które stabilizuje i koryguje trajektorię. Dzięki temu możliwe jest stopniowe budowanie mapy otoczenia w sposób inkrementalny, bez konieczności stosowania dodatkowych czujników. 
Ostatecznie prowadzi to do uzyskania spójnej reprezentacji przestrzeni na podstawie pojedynczej kamery, co znacząco upraszcza cały system przy zachowaniu użytecznej dokładności.

