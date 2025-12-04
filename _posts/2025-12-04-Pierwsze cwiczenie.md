---
layout: post
title: "Skalibrowana mata i pasy 3D — testy dokładności"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Pierwsza seria ćwiczeń. Przejazdy po torze modułowym i weryfikacja szacowania pozycji <!--more-->


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


#### Koncepcja. Środowisko do badań - składana mata i pasy ruchu
Ten post to propozycja zestawu ćwiczeń ilustrujących estymację trajektorii z przejazdu po makiecie drogi. Najpierw prezentuję samo środowisko. Makieta to czarna, składana mata oraz białe „klocki–szyny” udające pasy ruchu, wykonane w technice druku 3D. Elementy łączę na wpusty („na klik”). Przygotowałem dwa zestawy łuków o znanych promieniach $R_{\mathrm{ICR}}$: 250 mm i 500 mm (segmenty po 15°) oraz dwa zestawy prostych (długości 100 i 200 mm). Pozwala to szybko zbudować różne trasy.

Rysunek poniżej pokazuje jedną z konfiguracji toru:

<img src="{{ 'assets/images/cwiczenie1/TorCw1.JPG' | relative_url }}" alt="TorCw1" style="width:75%; max-width:100%; height:auto;" />

Ponieważ mata składa się z płytek o stałych wymiarach, a łuki i proste mają zadaną geometrię, traktuję trasę jako skalibrowaną. Na tak przygotowanej makiecie sprawdzam funkcjonalność modelu pojazdu i przydatność modeli obliczeniowych. Na ostrych zakrętach jadę wolno, aby ograniczyć poślizgi (pojazd nie ma dyferencjału kątowego). Poniżej surowe serie: prędkości i prądy silników oraz kąty serw (FBK) przed obróbką — dane wejściowe do resamplingu 50 Hz.

<img src="{{ 'assets/images/cwiczenie1/DaneSur.png' | relative_url }}" alt="DaneSur" style="width:100%; max-width:100%; height:auto;" />

Następnie, jak wcześniej, wyznaczam sygnały sterujące w punkcie środka pojazdu:

<img src="{{ 'assets/images/cwiczenie1/Sterowanie.png' | relative_url }}" alt="Sterowanie" style="width:100%; max-width:100%; height:auto;" />

Otrzymana trajektoria:

<img src="{{ 'assets/images/cwiczenie1/Trajekt.png' | relative_url }}" alt="Trajekt" style="width:100%; max-width:100%; height:auto;" />

Widać rozbieżność końcowej pozycji w kierunku osi \(Y\) (w \(X\) jej nie ma — pojazd rzeczywiście zatrzymał się nieco dalej). Różnica w \(Y\) wynosi około 10 cm. Jest to „znoszenie”, którego prawdopodobną przyczyną jest brak dyferencjału kątowego i wynikające z tego poślizgi na łukach; przy większej prędkości odchylenie byłoby zapewne większe. Na koniec prezentuję oszacowane błędy (przy założeniu normalnych rozkładów szumów):

<img src="{{ 'assets/images/cwiczenie1/Blad.png' | relative_url }}" alt="Blad" style="width:100%; max-width:100%; height:auto;" />

Błędy narastają powoli i przewidywalnie wzdłuż trasy, co potwierdza stabilność odometrii przy niskich prędkościach i umiarkowanych kątach skrętu.

#### Cel pierwszej serii ćwiczeń

- Zastosować odometrię 4WS (Ackermann‑predict) na danych z przejazdów.
- Przećwiczyć obróbkę logów NDJSON: import, synchronizacja, konwersje jednostek.
- Wyznaczyć trajektorię w układzie globalnym i oszacować niepewność (elipsy 3‑sigma).
- Zweryfikować wyniki na prostej makiecie z pasów (mata + wydruki 3D).

Instrukcja do tej serii ćwiczeń (pdf) jest na stronie githuba.


#### Materiały wejściowe
- Pliki z logami NDJSON: `out_00.txt`, `out_01.txt`. Przygotowane wstępnie pliki będą się znajdowały na serwerze githuba. 
- Parametry pojazdu: L = 260 mm, D = 153 mm, Rw = 37.25 mm, |δ| ≤ 35°.
- Skrypty startowe (MATLAB/Python): do parsowania, resamplingu, odometrii i rysowania wykresów.

#### 1) Import, synchronizacja, jednostki
Zadanie:
- Wczytaj log NDJSON linia po linii, rozdziel rekordy MOTOR i SERVO. 
- Przekonwertuj jednostki:
  - prędkości kół: spd [0.1 rpm] → v_wheel [m/s],
  - kąty serw: deg → rad (kanały servo_1, servo_2).
- Wyrównaj do 50 Hz (ΔT = 0.02 s) interpolacją liniową.
- Zdefiniuj sygnały modelu:
  - $V_k$ — średnia prędkość pojazdu,
  - $\delta_k$ — średnia z FBK po konwersji do rad.

Wyniki:
- wykresy $V(t)$, $\delta(t)$,
- krótki opis resamplingu (2–3 zdania).

#### 2) Odometria 4WS (Ackermann‑predict)

Równania aktualizacji stanu:

$$
\begin{aligned}
x_k = x_{k-1} + V_k\,\Delta T \cos \Theta_{k-1}
\end{aligned}
$$

$$
\begin{aligned}
y_k = y_{k-1} + V_k\,\Delta T \sin \Theta_{k-1}
\end{aligned}
$$

$$
\begin{aligned}
\Theta_k = \Theta_{k-1} + \dfrac{2 V_k\,\Delta T}{L}\,\tan \delta_k
\end{aligned}
$$

Zadanie:
- Przyjmij $x_0 = 0, y_0 = 0, \Theta_0 = 0 $.
- Narysuj trajektorię (x, y) z zaznaczonym początkiem i końcem.

Wyniki:
- wykres trajektorii,
- komentarz: wpływ znaku $\delta$ na stronę skrętu i ICR.


#### 3) Niepewność i elipsy 3‑sigma

Założenia:

$$
\begin{aligned}
P_0 = \mathrm{diag}\big(\sigma_x^2,\ \sigma_y^2,\ \sigma_\Theta^2\big)
\end{aligned}
$$

$$
\begin{aligned}
Q_0 = \mathrm{diag}\big(\sigma_V^2,\ \sigma_\delta^2\big)
\end{aligned}
$$

Macierze Jacobiego:

$$
\begin{aligned}
F_k =
\begin{bmatrix}
1 & 0 & -V_k\,\Delta T\,\sin \Theta_{k-1} \\
0 & 1 & \ \ V_k\,\Delta T\,\cos \Theta_{k-1} \\
0 & 0 & 1
\end{bmatrix}
\end{aligned}
$$

$$
\begin{aligned}
G_k =
\begin{bmatrix}
\Delta T \cos \Theta_{k-1} & 0 \\
\Delta T \sin \Theta_{k-1} & 0 \\
\dfrac{2\,\Delta T}{L}\,\tan \delta_k & \dfrac{2\,V_k\,\Delta T}{L}\,\sec^2 \delta_k
\end{bmatrix}
\end{aligned}
$$

Propagacja:
$\hat x_k^{-} = f(\hat x_{k-1}, z_k)$
$P_k^{-} = F_k P_{k-1} F_k^\top + G_k Q_k G_k^\top$
$z_k = [V_k,\ \delta_k]^\top$

Zadanie:
- Przyjmij np. $\sigma_V = 0.02,\mathrm{m/s}$, $\sigma_\delta = 0.5^\circ$ (w rad).
- Narysuj elipsy 3‑sigma co 10 s wzdłuż trajektorii.

Wyniki:
- trajektoria z elipsami,
- krótki komentarz: gdzie niepewność rośnie szybciej i dlaczego.

#### 4) Weryfikacja na makiecie: korytarz z pasów

Zadanie:
- Ułóż tor prosty i łagodny zakręt.
- Zrób przejazd prosto i po łagodnym łuku.
- Policz błąd boczny trajektorii względem osi korytarza.

Wyniki:
- wykres błędu bocznego vs. czas,
- interpretacja wpływu $V$ i $\delta$.

#### 5) Pętla zamknięta (ósemka / pętla parkingowa)

Zadanie:
- Ułóż pętlę, wykonaj przejazd.
- Policz wektor domknięcia: |$\Delta x$|, |$\Delta y$|, |$\Delta \Theta$|.

Wyniki:
- trajektoria + wektor domknięcia,
- komentarz: co dominuje w błędzie.

#### 6) (Opcjonalnie) Dopasowanie łuku

Zadanie:
- Na segmencie stałego skrętu dopasuj okrąg LS i porównaj promień z $R_c = L/(2 \tan \bar{\delta})$.

Wyniki:
- promień z dopasowania vs. teoretyczny,
- błąd względny [%].

---

#### Kryteria oceniania

Poprawność techniczna
- Import NDJSON, rozdzielenie MOTOR/SERVO.
- Konwersje jednostek poprawne.
- Resamplowanie 50 Hz opisane i poprawne.
- Implementacja odometrii 4WS zgodna ze wzorami.
- $F_k, G_k$ policzone dobrze; $Q_k, P_0$ uzasadnione.

Jakość analizy
- Czytelne trajektorie z oznaczeniami.
- Elipsy 3‑sigma poprawnie zorientowane.
- Błąd boczny obliczony; wnioski sensowne.
- Wektor domknięcia policzony i omówiony.
- Wykrywanie anomalii i wpływ filtracji.

Prezentacja i wnioski
- Podpisane osie, jednostki, legendy.
- Zwięzłe komentarze pod wykresami.
- Wnioski: gdzie i dlaczego rośnie niepewność; kiedy potrzebna korekcja.

Punktacja sugerowana
- Zad. 1–2: 30%
- Zad. 3: 25%
- Zad. 4: 15%
- Zad. 5: 15%
- Zad. 6: 10%
- Prezentacja/wnioski: 5%

Ocena 3 - 51%, 3.5 - 61%, 4 - 71%, 4.5 - 81%, 5 ponad 91%. 

Uwaga praktyczna
- W tej implementacji krok „predict” opiera się na pomiarach $V, \delta$ (FBK), nie na komendach, aby uniknąć błędów od opóźnień aktuatorów.






