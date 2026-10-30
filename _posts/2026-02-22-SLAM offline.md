---
layout: post
title: "SLAM offline"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Kiedy trajektoria wreszcie może się poprawić?<!--more-->


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


### 1) SLAM (Simultaneous Localization And Mapping)

W poprzednim wpisie, dotyczącym map fittingu, celowo zamroziłem trajektorię odometryczną. Nie zakładałem, że jest poprawna. Chciałem sprawdzić, jak spójna będzie mapa nieruchomych landmarków, jeśli nie pozwolę algorytmowi korygować położenia pojazdu.
Wyniki pokazały, że kolejne obserwacje tego samego markera nie zawsze trafiają w jedno miejsce na mapie. Źródłem rozbieżności mogą być błędy odometrii, lokalnych pomiarów kamerą lub obu tych elementów.
Teraz przechodzę do SLAM-u (Simultaneous Localization And Mapping), czyli jednoczesnej estymacji trajektorii pojazdu i mapy otoczenia. Różnica względem map fittingu jest zasadnicza: trajektoria nie jest już zamrożona (rys. 1). Algorytm może zmieniać pozycje i orientacje pojazdu, tak aby uzgodnić pomiary odometryczne z obserwacjami nieruchomych landmarków.
W tym eksperymencie wykorzystuję SLAM offline. Pracuję na zapisanych wcześniej danych z przejazdu, więc optymalizacja nie musi działać w czasie rzeczywistym.
Chcę sprawdzić, na ile dopuszczenie korekty trajektorii zmniejszy niespójność obserwacji landmarków i czy otrzymana mapa będzie bliższa rzeczywistej geometrii stanowiska. To dwa różne pytania, dlatego oprócz wyników optymalizacji potrzebne będzie również porównanie z Ground Truth.

<img src="{{ 'assets/images/SLAMoffline/schemat SLAM.png' | relative_url }}" alt="schemat SLAM" style="width:100%; max-width:100%; height:auto;" />

*Rys. 1. Miejsce map fittingu i SLAM-u w procesie tworzenia mapy landmarków. Map fitting wykorzystuje trajektorię odometryczną jako ustaloną, natomiast SLAM pozwala później wspólnie optymalizować trajektorię i mapę.*


## 2) Ten sam marker w kilku miejscach mapy
W poprzednim eksperymencie pokazałem, że kolejne obserwacje tego samego nieruchomego markera ArUco, po przeliczeniu do globalnego układu współrzędnych, nie zawsze trafiają w jedno miejsce. Przy zamrożonej trajektorii odometrycznej mogłem jedynie wyznaczyć średnią pozycję landmarku i zmierzyć rozrzut jego obserwacji.
W SLAM-ie tę samą informację wykorzystuję inaczej. Położenie markera w świecie jest wspólne dla wszystkich jego obserwacji, natomiast pozycje i orientacje pojazdu mogą się zmieniać podczas optymalizacji.
Jeżeli samochód obserwuje marker z różnych miejsc, każda obserwacja nakłada ograniczenie na wzajemne położenie pojazdu i landmarku. Algorytm musi więc znaleźć takie pozycje pojazdu i markerów, które możliwie dobrze uzgodnią te pomiary, nie odchodząc przy tym nadmiernie od ruchu zarejestrowanego przez odometrię.
Szczególnie przydatne są ponowne obserwacje tych samych markerów po przejechaniu pętli. Łączą one fragmenty trajektorii, które są odległe w czasie, ale odnoszą się do tych samych nieruchomych punktów. Dzięki temu mogą ograniczać narastającą niespójność położenia.
Nie oznacza to jeszcze, że SLAM odtworzy rzeczywistą trajektorię. Wynik zależy od jakości pomiarów oraz przyjętych ograniczeń. Możemy jednak sprawdzić, czy wspólna optymalizacja pozwala uzyskać bardziej spójną trajektorię i mapę niż przy zamrożonej odometrii.

## 3) Minimalna teoria

W chwili $k$ pojazd ma pozę w układzie globalnym:

$$
\mathbf{x}_k=
\begin{bmatrix}
x_k & y_k & \varphi_k
\end{bmatrix}^{T}
$$

Z poprzedniego etapu przetwarzania obrazu otrzymuję lokalną obserwację markera $j$, wyrażoną w metrach względem pojazdu:

$$
\mathbf{z}_{k,j}^{\mathrm{local}}=
\begin{bmatrix}
Y_{\mathrm{BEV}}\\
-X_{\mathrm{BEV}}
\end{bmatrix}
$$

Geometria kamery względem pojazdu została już uwzględniona podczas przygotowania BEV. W SLAM-ie nie optymalizuję dodatkowych parametrów tej transformacji.
Znając pozę pojazdu, mogę przeliczyć obserwację markera do układu globalnego:

$$
\hat{\mathbf{p}}_{k,j}^{G}=
\begin{bmatrix}
x_k\\
y_k
\end{bmatrix}
+
R(\varphi_k)\mathbf{z}_{k,j}^{\mathrm{local}}
$$

gdzie $\(R(\varphi_k)\)$ jest macierzą obrotu wynikającą z orientacji pojazdu.

W mapie marker o identyfikatorze $j$ ma jedną estymowaną pozycję $\mathbf{p}_j^G$. Różnicę pomiędzy pozycją wynikającą z obserwacji a pozycją w mapie opisuję residualem:

$$
\mathbf{r}_{\mathrm{obs},k,j}
=
\hat{\mathbf{p}}_{k,j}^{G}-\mathbf{p}_j^G
$$

W aktualnym programie residual obserwacji obliczam równoważnie w lokalnym układzie pojazdu. Porównuję zmierzone położenie markera z położeniem przewidywanym na podstawie aktualnej pozy pojazdu i mapy:

$$
\mathbf{r}_{\mathrm{obs},k,j}^{\mathrm{local}}
=
R(\varphi_k)^T
\left(
\mathbf{p}_j^G-
\begin{bmatrix}x_k\\y_k\end{bmatrix}
\right)
-
\mathbf{z}_{k,j}^{\mathrm{local}}
$$

Obie postacie mają tę samą długość residualu, ponieważ różnią się jedynie obrotem.
Drugim rodzajem ograniczenia jest odometria. Dla kolejnych węzłów trajektorii znam względny ruch wynikający z odometrii, oznaczony jako $\Delta\mathbf{x}_k^{\mathrm{odom}}$. Z aktualnie estymowanych poz mogę obliczyć analogiczny ruch $\Delta\mathbf{x}_k^{\mathrm{SLAM}}$.

Residual odometryczny ma zatem postać:

$$
\mathbf{r}_{\mathrm{odom},k}
=
\Delta\mathbf{x}_k^{\mathrm{SLAM}}
-
\Delta\mathbf{x}_k^{\mathrm{odom}}
$$

Różnicę orientacji sprowadzam do przedziału od $-\pi\)$ do $\(\pi$, aby poprawnie uwzględnić okresowość kąta.
Algorytm szuka takich poz pojazdu i pozycji landmarków, które zmniejszają obie grupy residuali. Nie są one jednak traktowane jednakowo. Residuale normalizuję przez przyjęte niepewności pomiarów: 30 mm dla translacji odometrycznej, 2° dla zmiany orientacji oraz 50 mm dla lokalnych obserwacji landmarków.
Zamiast zwykłej sumy kwadratów wykorzystuję funkcję straty Huber, która ogranicza wpływ dużych residuali. Do funkcji celu dodaję również ograniczenie pierwszej pozy pojazdu, czyli anchor, aby ustalić układ odniesienia.

Algorytm minimalizuje łączny koszt wynikający z residuali odometrii,
obserwacji landmarków oraz ograniczenia pierwszej pozy pojazdu (anchor):

$$
\min_{\{\mathbf{x}_k\},\{\mathbf{p}_j^G\}}
\frac{1}{2}
\sum_{i=1}^{N_r}
\rho\left(\widetilde{r}_i^2\right)
$$

gdzie:

- $\widetilde{r}_i$ – pojedyncza składowa znormalizowanego wektora residuali,
- $N_r$ – całkowita liczba składowych residuali,
- $\rho$ – funkcja straty Huber.

Wektor residuali zawiera błędy względnego ruchu pojazdu, błędy
lokalnych obserwacji landmarków oraz ograniczenie pierwszej pozy.
Poszczególne składowe są dzielone przez przyjęte niepewności pomiarów.

Dzięki funkcji Huber duże residuale mają mniejszy wpływ na wynik
niż przy zwykłej metodzie najmniejszych kwadratów.

W ten sposób SLAM szuka kompromisu między ruchem wynikającym z odometrii a położeniami nieruchomych markerów. W przeciwieństwie do map fittingu może przy tym zmieniać całą trajektorię.

<img src="{{ 'assets/images/SLAMoffline/graf SLAM.png' | relative_url }}" alt="graf SLAM" style="width:100%; max-width:100%; height:auto;" />

*Rys. 2. Schemat optymalizacji SLAM. Odometria i lokalne obserwacje landmarków dostarczają ograniczeń, na podstawie których wyznaczane są trajektoria pojazdu i mapa. Ground Truth służy wyłącznie do późniejszej walidacji.*

### 4) Dwie informacje, które trzeba pogodzić
Najprościej myśleć o SLAM-ie jako o próbie pogodzenia dwóch informacji.
Odometria mówi, o ile samochód przemieścił się i obrócił pomiędzy kolejnymi chwilami. Nie daje jednak pewności, że wszystkie te przemieszczenia, zsumowane na całej trasie, prowadzą do prawidłowej pozycji pojazdu.
Landmarki dostarczają innego rodzaju informacji. Marker o danym ID pozostaje nieruchomy, więc niezależnie od tego, ile razy go zaobserwuję, powinien mieć jedną pozycję w układzie globalnym. Jeśli obserwacje nie są zgodne, oznacza to, że trajektoria i mapa nie pasują do siebie.
SLAM próbuje uzgodnić oba źródła pomiarów. Korekta jednej pozy pojazdu może poprawić zgodność z obserwowanym markerem, ale jednocześnie pogorszyć zgodność ruchu z odometrią. Dlatego algorytm nie może dowolnie przesuwać pojazdu, aby idealnie dopasować każdy landmark.
Znaczenie mają tutaj przyjęte niepewności pomiarów. W aktualnym programie wynoszą one 30 mm dla translacji odometrycznej, 2° dla zmiany orientacji oraz 50 mm dla lokalnej obserwacji landmarku. Są to parametry modelu optymalizacji, a nie dokładności potwierdzone niezależnym eksperymentem. Ich wartości wpływają na kompromis pomiędzy odometrią i kamerą.
Pozostaje jeszcze kwestia układu odniesienia. Gdyby przesunąć i obrócić jednocześnie całą trajektorię oraz wszystkie landmarki, wzajemne relacje między nimi pozostałyby takie same. Optymalizacja nie potrafiłaby więc samodzielnie ustalić jednego, bezwzględnego położenia całego rozwiązania.
Aby usunąć tę swobodę, przyjmuję pierwszą pozę odometryczną jako punkt odniesienia, czyli anchor. Jej pozycja i orientacja są silnie związane z wartościami początkowymi. Nie oznacza to, że pierwsza poza jest znana idealnie. Ustala jedynie układ współrzędnych, w którym wyznaczam trajektorię i mapę.
Ostatecznie SLAM może poprawiać zarówno trajektorię, jak i pozycje landmarków, ale musi przy tym respektować ograniczenia wynikające z odometrii, obserwacji kamery i przyjętego punktu odniesienia.


### 5) Budowa grafu i optymalizacja
W praktyce SLAM offline rozpoczyna się od przygotowania danych. Korzystam z dwóch plików CSV: `synced_with_traj.csv`, zawierającego zapisane pozy odometryczne pojazdu, oraz `landmarks_local.csv`, w którym znajdują się lokalne współrzędne wykrytych markerów ArUco. Oba zbiory łączy identyfikator klatki `frame_id`.
Nie wykorzystuję wszystkich klatek nagrania. Z klatek zawierających obserwacje markerów wybieram co trzecią. Następnie odrzucam landmarki, które mają mniej niż trzy obserwacje. Dzięki temu ograniczam wielkość problemu optymalizacji, zachowując powtarzające się pomiary tych samych punktów.
Każda wybrana poza pojazdu tworzy węzeł grafu. Sąsiednie węzły łączą ograniczenia odometryczne, opisujące zmierzony ruch względny. Obserwacje ArUco tworzą natomiast połączenia pomiędzy pozami pojazdu a landmarkami. Ten sam marker może być związany z wieloma pozami, również z różnych fragmentów przejazdu.
Punktem początkowym optymalizacji jest trajektoria odometryczna. Początkowe położenia landmarków wyznaczam metodą map fittingu: każdą lokalną obserwację przeliczam do układu globalnego z wykorzystaniem odometrii, a następnie obliczam średnią pozycję dla danego ID.
Z tak przygotowanego grafu tworzę wektor zmiennych obejmujący pozycje i orientacje pojazdu oraz współrzędne landmarków. Względne pomiary odometryczne i lokalne obserwacje kamery pozostają niezmienione. Podczas optymalizacji zmieniają się estymowane pozy i mapa.
Do rozwiązania wykorzystuję funkcję `least_squares` z biblioteki SciPy, metodę TRF oraz funkcję straty Huber. Algorytm minimalizuje residuale opisane w poprzedniej sekcji, uwzględniając ich przyjęte niepewności i ograniczenie pierwszej pozy pojazdu.
Po zakończeniu optymalizacji otrzymuję skorygowaną trajektorię oraz mapę landmarków, zapisywane odpowiednio w plikach `slam_poses.csv` i `slam_landmarks.csv`.
Na tym etapie nie zmieniam geometrii BEV ani nie estymuję parametrów modelu ruchu. Sprawdzam, jaki wynik można uzyskać, korygując samą trajektorię i pozycje landmarków przy ustalonych pomiarach wejściowych.

<img src="{{ 'assets/images/SLAMoffline/algorytm SLAM.png' | relative_url }}" alt="algorytm SLAM" style="width:100%; max-width:100%; height:auto;" />

*Rys. 3. Przebieg algorytmu SLAM offline: przygotowanie danych, inicjalizacja na podstawie odometrii i map fittingu oraz wspólna optymalizacja trajektorii i mapy landmarków.*

### 6) Test 1 – trasa z ostrymi zakrętami
Pierwszy eksperyment przeprowadziłem na krótkiej trasie z ostrymi zakrętami, znanej już z poprzedniego wpisu o map fittingu. Samochód wykonał dwa przejazdy po tym samym torze, dzięki czemu algorytm mógł wykorzystać ponowne obserwacje nieruchomych markerów.
Graf SLAM zawierał 366 węzłów trajektorii, 12 landmarków oraz 366 obserwacji markerów. Punktem początkowym była trajektoria odometryczna i mapa landmarków wyznaczona metodą map fittingu.
Przed optymalizacją RMS obserwacji landmarków wynosił 264,7 mm. Po SLAM zmniejszył się do 6,2 mm, czyli o około 97,7%. Oznacza to, że algorytm uzgodnił obserwacje markerów z wyznaczoną trajektorią i mapą znacznie lepiej niż w rozwiązaniu początkowym.

<img src="{{ 'assets/images/SLAMoffline/SlamCalib.png' | relative_url }}" alt="SlamCalib" style="width:100%; max-width:100%; height:auto;" />

*Rys. 4 Trajektoria odometryczna i wyznaczona trajektoria po SLAM wraz z położeniem landmarków*

Zmiana nie była jednak niewielka. Średnia odległość pomiędzy pozycjami odometrycznymi a odpowiadającymi im pozycjami SLAM wyniosła 452,2 mm, a maksymalna 877,2 mm. Nie oznacza to, że odometria miała dokładnie takie błędy. Są to wartości korekt wprowadzonych przez optymalizację, a nie odległości od rzeczywistej trajektorii.
Jednocześnie RMS residualu odometrycznego pomiędzy kolejnymi węzłami wyniósł 5,6 mm dla translacji oraz 0,19° dla orientacji. SLAM zmienił więc globalny przebieg trajektorii, zachowując stosunkowo niewielkie różnice w ruchu względnym pomiędzy sąsiednimi pozami.
Wynik pokazuje, że dopuszczenie korekty trajektorii pozwala znacznie zmniejszyć niespójność obserwacji landmarków. Na tym etapie nie wiadomo jednak, czy otrzymana mapa odpowiada rzeczywistemu rozmieszczeniu markerów. Do takiej oceny potrzebne jest porównanie z Ground Truth.

## 7) Test 2 – trasa z łagodniejszymi zakrętami

Drugi eksperyment przeprowadziłem na dłuższej trasie z łagodniejszymi zakrętami. Podobnie jak wcześniej samochód wykonał dwa przejazdy po tym samym torze. Sposób przetwarzania danych i parametry optymalizacji pozostały niezmienione.
Graf SLAM zawierał 599 węzłów trajektorii, 20 landmarków i 599 obserwacji markerów. Przed optymalizacją RMS obserwacji landmarków wynosił 243,7 mm, a po SLAM zmniejszył się do 111,2 mm, czyli o około 54,4%.

<img src="{{ 'assets/images/SLAMoffline/SLAM przejazd 2.png' | relative_url }}" alt="SLAM przejazd 2" style="width:100%; max-width:100%; height:auto;" />

*Rys.5: Trajektoria odometryczna i wyznaczona trajektoria po SLAM wraz z położeniem landmarków dla drugiego eksperymentu.*

Średnia korekta położenia pojazdu względem odometrii wyniosła 285,5 mm, a maksymalna 736,8 mm. Residuale odometryczne pozostały niewielkie: RMS wyniósł 2,9 mm dla translacji oraz 0,15° dla orientacji.
W porównaniu z pierwszym eksperymentem SLAM wprowadził mniejsze korekty trajektorii. Jednocześnie znacznie mniejsza była poprawa zgodności obserwacji landmarków. Na trasie z ostrymi zakrętami RMS zmniejszył się z 264,7 mm do 6,2 mm, natomiast tutaj z 243,7 mm do 111,2 mm.
Różnica jest zastanawiająca, ponieważ w obu eksperymentach wykorzystałem ten sam algorytm i te same parametry optymalizacji. Nie oznacza to jednak, że SLAM działa gorzej na łagodniejszych zakrętach. Trasy różnią się długością, liczbą landmarków i strukturą obserwacji, co może wpływać na wynik.
Warto również pamiętać, że RMS obserwacji jest miarą spójności rozwiązania z lokalnymi pomiarami kamery. Niewielka wartość nie gwarantuje poprawnej geometrii mapy, podobnie jak większy RMS nie musi oznaczać większego błędu położenia landmarków względem rzeczywistości.
Dlatego na podstawie samych wyników optymalizacji nie rozstrzygam, który eksperyment dał dokładniejszą mapę. W następnej sekcji porównam pozycje landmarków z ich rzeczywistym rozmieszczeniem na stanowisku, czyli Ground Truth.

### 8) SLAM a Ground Truth – czy spójna mapa jest poprawna?
Wyniki poprzednich eksperymentów pokazują, że SLAM potrafi wyraźnie zmniejszyć rozrzut obserwacji landmarków. Nie oznacza to jednak, że wyznaczone pozycje markerów odpowiadają ich rzeczywistemu rozmieszczeniu.
Aby to sprawdzić, porównałem mapy otrzymane ze SLAM z Ground Truth (GT), czyli zmierzonymi pozycjami markerów na stanowisku. Pozycje GT nie były wykorzystywane podczas optymalizacji SLAM. Służą wyłącznie do oceny otrzymanych wyników.
Problem polega na tym, że mapa SLAM i GT mogą być zapisane w różnych układach współrzędnych. Dlatego przed obliczeniem błędów należy je wyrównać. Zastosowałem trzy sposoby dopasowania:

- Rigid (SE(2)) – dopuszcza obrót i przesunięcie, ale nie zmienia skali ani kształtu mapy.

- Similarity – oprócz obrotu i przesunięcia dopuszcza jednolitą zmianę skali.

- Affine – pozwala również na różne skalowanie w poszczególnych kierunkach i odkształcenie układu.

Pierwsze dopasowanie służy ocenie zgodności geometrii mapy bez jej deformowania. Pozostałe dwa traktuję diagnostycznie: pozwalają sprawdzić, jaką część rozbieżności można wyjaśnić zmianą skali lub bardziej ogólnym przekształceniem geometrycznym.

Wyniki porównania przedstawia tabela.

**Tabela 1. Porównanie wyników SLAM z Ground Truth dla obu eksperymentów.**

| Miara | Test 1 – ostre zakręty | Test 2 – łagodne zakręty |
|---|---:|---:|
| RMS obserwacji po SLAM [mm] | 6,2 | 111,2 |
| Rigid RMSE względem GT [mm] | 165,3 | 106,0 |
| Similarity RMSE względem GT [mm] | 105,9 | 71,6 |
| Affine RMSE względem GT [mm] | 89,0 | 34,8 |
| Skala similarity [-] | 1,1991 | 1,0468 |
| Odchylenie standardowe skali par landmarków [-] | 0,1311 | 0,0332 |


Wynik jest interesujący. W pierwszym eksperymencie RMS obserwacji po SLAM wyniósł zaledwie 6.2 mm, ale błąd mapy względem GT po sztywnym wyrównaniu pozostał na poziomie 165.3 mm. W drugim eksperymencie RMS obserwacji był znacznie większy, natomiast pozycje landmarków okazały się bliższe GT: rigid RMSE wyniósł 106.0 mm.
Oznacza to, że wewnętrzna spójność obserwacji nie jest wystarczającą miarą dokładności mapy.


<div style="display:flex; gap:10px; width:100%; align-items:center;">

  <img
    src="{{ 'assets/images/SLAMoffline/slam vs GT 1.png' | relative_url }}"
    alt="slam vs GT 1"
    style="flex:1 1 0; width:100%; min-width:0; height:auto;"
  />

  <img
    src="{{ 'assets/images/SLAMoffline/slam vs GT 2.png' | relative_url }}"
    alt="slam vs GT 2"
    style="flex:1 1 0; width:100%; min-width:0; height:auto;"
  />

</div>

*Rys. 6: mapy landmarków po sztywnym wyrównaniu do GT dla obu eksperymentów*

Warto również sprawdzić, co zmieniła sama optymalizacja SLAM. W tabeli poniżej porównuję błędy map początkowych, wyznaczonych z odometrii, z błędami map po SLAM.

**Tabela 2. Porównanie zgodności map landmarków z Ground Truth przed i po optymalizacji SLAM.**

| Miara | Test 1 – ostre zakręty (przed → po SLAM) | Test 2 – łagodne zakręty (przed → po SLAM) |
|---|---:|---:|
| Rigid RMSE [mm] | 176,3 → 165,3 | 109,3 → 106,0 |
| Similarity RMSE [mm] | 162,9 → 105,9 | 100,1 → 71,6 |
| Affine RMSE [mm] | 140,8 → 89,0 | 96,3 → 34,8 |
| Odchylenie standardowe skali par landmarków [-] | 0,2627 → 0,1311 | 0,0683 → 0,0332 |


W obu eksperymentach poprawa po sztywnym wyrównaniu jest stosunkowo niewielka. Znacznie większą różnicę widać po dopuszczeniu zmiany skali i przekształcenia afinicznego.
Dodatkowa diagnostyka odległości pomiędzy parami markerów pokazała, że po SLAM zmniejszył się rozrzut współczynników skali. Mapa stała się więc bardziej jednorodna pod względem relacji odległości, mimo że nadal różni się od GT.
W teście pierwszym dopasowanie similarity wymagało współczynnika skali około 1,20, natomiast w drugim około 1,05. Nie oznacza to bezpośrednio takiego samego błędu skali BEV. Współczynnik ten opisuje łączną różnicę pomiędzy geometrią uzyskanej mapy a GT.
Przeprowadziłem również analizę leave-one-out, w której kolejno wykluczałem jeden landmark i ponownie dopasowywałem pozostałe punkty do GT. W żadnym z eksperymentów usunięcie pojedynczego markera nie spowodowało radykalnego zmniejszenia błędu. Rozbieżność dotyczy więc geometrii całej mapy, a nie jednego błędnie umieszczonego punktu.
Wyniki pokazują ograniczenie obecnego rozwiązania. SLAM poprawia zgodność obserwacji i część relacji geometrycznych pomiędzy landmarkami, ale nie gwarantuje odtworzenia ich rzeczywistych pozycji.
Nie potrafię jeszcze rozstrzygnąć, jaka część pozostałych błędów wynika z odometrii, a jaka z lokalnych pomiarów kamerą. Do tego potrzebna będzie niezależna weryfikacja dokładności BEV, bez wykorzystywania trajektorii odometrycznej.

## 9) Co dalej?

SLAM pozwolił mi zmniejszyć niespójność obserwacji landmarków i skorygować trajektorię odometryczną. Porównanie z Ground Truth pokazało jednak, że mapa może być wewnętrznie spójna, a mimo to wyraźnie różnić się od rzeczywistej geometrii stanowiska.
Szczególnie interesująca okazała się różnica pomiędzy dwoma eksperymentami. Na trasie z ostrymi zakrętami uzyskałem bardzo mały RMS obserwacji po SLAM, ale większy błąd mapy względem GT. Na dłuższej trasie z łagodniejszymi zakrętami było odwrotnie. To pokazuje, że sam residual optymalizacji nie wystarcza do oceny dokładności lokalizacji.
Nie wiadomo jeszcze, w jakim stopniu za pozostałe rozbieżności odpowiada odometria, a w jakim lokalne pomiary z kamery. Oba źródła informacji są wykorzystywane jednocześnie, dlatego trudno ocenić ich dokładność wyłącznie na podstawie końcowego wyniku SLAM.
W kolejnym etapie chcę więc wrócić do geometrii BEV. Zamierzam porównać pozycje i odległości markerów wyznaczone bezpośrednio z obrazu z ich zmierzonymi wartościami. Pozwoli to sprawdzić dokładność przekształcenia obrazu do układu pojazdu bez udziału odometrii.
Dopiero mając taką informację, będzie można rozsądniej ocenić wpływ odometrii i przyjętego modelu ruchu na wyniki SLAM. Chciałbym również sprawdzić, jak zmiana wag pomiarów wpływa na kompromis pomiędzy zgodnością obserwacji a geometrią otrzymanej mapy.
Na razie najważniejszy wniosek z tych eksperymentów jest prosty: SLAM może uzgodnić obserwacje i trajektorię, ale nie gwarantuje, że uzyskana mapa odpowiada rzeczywistości. Do oceny jej dokładności potrzebny jest niezależny punkt odniesienia.