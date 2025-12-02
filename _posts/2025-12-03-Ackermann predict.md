---
layout: post
title: "Jak rośnie błąd trajektorii? Ackermann‑predict w praktyce"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Rachunek niepewności i propagacja błędu w modelu ruchu (Ackermann‑predict) <!--more-->

### 1) Koncepcja obliczania błędów
Zakładam, że źródłem odchyleń od rzeczywistych nieznanych wartości zmiennych są szumy (procesu i pomiaru). Obliczenia prowadzę iteracyjnie w czasie dyskretnym. Ruch opisuję kinematyką 4WS (przeciwfazowo): przemieszczeniem po łuku i obrotem wokół ICR. Model wyjściowy to równania kroku aktualizacji czasu, wiec je tu przypomnę:

$$
x_k = x_{k-1} + V_k\,\Delta T \cos \Theta_{k-1}
$$

$$
y_k = y_{k-1} + V_k\,\Delta T \sin \Theta_{k-1}
$$

$$
\Theta_k = \Theta_{k-1} + \dfrac{2 V_k\,\Delta T}{L}\,\tan \delta_k
$$

Chcemy znać, jak rośnie niepewność stanu między chwilami k−1 i k. Błąd stanu w k−1 opisuje macierz kowariancji $P_{k-1}$. Naszym celem jest wyznaczyć $P_k$.

#### 2) Błędy stanu i sterowania
Przyjmuję, że rozkłady gęstości prawdopodobieństw błędów są normalne. Niepewność początkową definiuję przez macierz kowariancji stanu:

\(P_0 = \mathrm{diag}\big(\sigma_x^2,\ \sigma_y^2,\ \sigma_\Theta^2\big)\)

Niepewność sterowania (wejść) opisuje macierz kowariancji:

\(Q_0 = \mathrm{diag}\big(\sigma_V^2,\ \sigma_\delta^2\big)\)

gdzie \(\sigma\) to odchylenie standardowe, a \(\sigma^2\) wariancja.

#### 3) Linearyzacja kroku (Ackermann‑predict)
Równania ruchu sa nieliniowe. Stosuję ich liniowe przybliżenie w otoczeniu punktu pracy \(\bar x,\ \bar u\) (Euler w przód, stałe sterowanie w kroku):

\(x_k \approx f(\bar x,\bar u) + F_k\,(x_{k-1}-\bar x) + G_k\,(u_k-\bar u)\)


\(x_k = [\,x_k,\ y_k,\ \Theta_k\,]^T\)

\(F_k = \dfrac{\partial f}{\partial x}\big|_{k-1}\)

\(F_k =
\begin{bmatrix}
1 & 0 & -V_k\,\Delta T\,\sin \Theta_{k-1} \\
0 & 1 & \ \ V_k\,\Delta T\,\cos \Theta_{k-1} \\
0 & 0 & 1
\end{bmatrix}\)


\(G_k =
\begin{bmatrix}
\Delta T \cos \Theta_{k-1} & 0 \\
\Delta T \sin \Theta_{k-1} & 0 \\
\dfrac{2\,\Delta T}{L}\,\tan \delta_k & \dfrac{2\,V_k\,\Delta T}{L}\,\sec^2 \delta_k
\end{bmatrix}\)

#### 4) Krok predykcji oparty na pomiarach, nie na sterowaniach
W mojej implementacji nie korzystam z sygnałów sterujących jako wejść do predykcji. Powód jest praktyczny: opóźnienia między wydaniem komendy a realną odpowiedzią serwa i silników są na tyle duże i zmienne, że „u” nie reprezentuje tego, co działo się z pojazdem w danym kroku czasu. Zamiast tego opieram krok przewidywania bezpośrednio na pomiarach prędkości i kąta skrętu (FBK z enkoderów serw i prędkości kół po konwersji do jednostek SI). Innymi słowy: to, co zwykle w literaturze bywa traktowane jako sterowanie, u mnie pełni rolę obserwowanej wielkości kinematycznej, już uwzględniającej wewnętrzne dynamiki aktuatorów i ich opóźnienia.

W tej konwencji:


- Równania aktualizacji stanu pozostają bez zmian, ale symbolicznie traktuję \(V_k\) i \(\delta_k\) jako pomiary (z niepewnością), a nie polecenia sterujące.

- Macierz kowariancji wejść \(Q_k\) opisuje niepewność pomiarów prędkości i kąta skrętu (szum/rozrzut FBK), a nie niepewność komend.

Zatem propagacja niepewności w kroku „predict” ma postać

\(\hat x_k^{-} = f(\hat x_{k-1},\, z_k)\),

\(P_k^{-} = F_k P_{k-1} F_k^\top + G_k Q_k G_k^\top\),

gdzie \(z_k = [V_k,\ \delta_k]^\top\) to wektor pomiarów kinematycznych (a nie sterowań).

Dlaczego tak?


Eliminuje to błąd modelowania związany z nieznanym i zmiennym opóźnieniem wykonawczym. Predykcja opiera się na tym, co rzeczywiście zaszło w pojeździe w danym kroku, a nie na intencji sterowania.


#### 5) O przyszłej fuzji
W kolejnych etapach projektu planuję dołożyć niezależne źródło informacji o położeniu z wizji (mapa wizualna/SLAM, w tym wariant monokularny). Gdy będę znał niepewności lokalizacji z mapy obrazów, połączę „trajektorię kinematyczną” (z enkoderów) z „trajektorią wizualną” metodą bayesowską, ważoną wiarygodnościami obu źródeł. Na tym etapie wystarczy świadomość, że obecny krok „predict” już uwzględnia niepewność pomiarów \(V,\delta\); dodatkowe czujniki wejdą później jako niezależne obserwacje tego samego stanu.

#### 6) Przejazd 1
Wynik estymacji błędu wzdłóż trajektorii przedstawia rys:

<img src="{{ 'assets/images/AckermannPredict/Ackermann1.png' | relative_url }}" alt="Ackermann1" style="width:100%; max-width:100%; height:auto;" />

Krzyżykami zaznaczam wybrane punkty trajektorii, w których prezentuję wynik estymaty stanu. Błędy pozycji liczone są w lokalnym układzie pojazdu (w punkcie środkowym), a ich rozrzut w kierunku wzdłużnym i poprzecznym do osi pojazdu przedstawia elipsa 3‑sigma. Niepewność orientacji (kursu) ilustruje czerwona strzałka: im dłuższa i „grubsza”, tym większy błąd kąta. Na wykresie widać, że błąd z czasem rośnie, ale robi to powoli i w przewidywalny sposób. Nie „rozjeżdża się” szybko. Dzięki temu nasza wyznaczona trajektoria pozostaje użyteczna przez dłuższy czas, nawet bez dodatkowych poprawek z innych czujników.

#### 6) Przejazd 2
Rysunek przedstawia wyniki szacowania błędów pozycjonowania i kursu dla przejazdu nr 2.

<img src="{{ 'assets/images/AckermannPredict/Ackermann2.png' | relative_url }}" alt="Ackermann2" style="width:100%; max-width:100%; height:auto;" />

Analiza potwierdza wcześniejsze wnioski: niepewność rośnie wzdłuż trasy stopniowo, a wartości pozostają umiarkowane w badanych warunkach prędkości i kątów skrętu.

#### 7) Wnioski
- Kinematyka 4WS (Ackermann‑predict oparta na pomiarach V, δ) pozwala stabilnie wyznaczać trajektorię bez dodatkowych czujników.
- Niepewność położenia i kursu narasta w czasie stopniowo; przy małych prędkościach i umiarkowanych kątach skrętu pozostaje niewielka.
- Elipsy 3‑sigma i wektory błędu kąta potwierdzają przewidywalny kierunek wzrostu niepewności: bardziej wzdłuż kierunku jazdy i w orientacji.
- Oparcie predykcji na pomiarach (a nie na komendach sterujących) eliminuje problem zmiennych opóźnień aktuatorów i poprawia spójność estymacji.
- Dla dłuższych odcinków przydatna będzie dodatkowa korekcja z niezależnego źródła (np. wizja/SLAM), łączona bayesowsko z odometrią, aby ograniczyć kumulację błędu.

#### 8) Co dalej
W kolejnym wpisie omówie propozycje ćwiczeń dydaktycznych bazujących na przedstawionym dotychczas materiale
