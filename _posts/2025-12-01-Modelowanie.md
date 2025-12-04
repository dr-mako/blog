---
layout: post
title: "Aktualizacja położenia w oparciu o model i wyznaczanie trajektorii ruchu"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Opis modelu <!--more-->

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


Parametry geometryczne prototypu

- Rozstaw kół: $D = 153\,\mathrm{mm}$
- Rozstaw osi: $L = 260\,\mathrm{mm}$
- Zakres skrętu mechaniczny: 0° - 90°, ograniczenie programowe: $35^\circ$
- Promień koła: $R_w = 37.25\,\mathrm{mm}$

### 1) Koncepcja
W tym poście, do opisu ruchu pojazdu zastosuję prosty model Ackermanna w ujęciu „rowerowym” (bicycle model dla konfiguracji 2WS przednia para kół skrętna), a następnie rozszerzę go do mojego przypadku 4WS (przód + tył skrętne, przeciwfazowo).

W modelu 2WS skrętna jest wyłącznie oś przednia, a oś tylna jest toczna. W idealizacji Ackermanna dopuszczamy różne kąty skrętu kół wewnętrznego i zewnętrznego na osi przedniej tak, aby przedłużenia ich płaszczyzn przecinały się w jednym punkcie ICR (środek obrotu — Instantaneous Center of Rotation). Dzięki temu ruch jest bezpoślizgowy: wszystkie koła poruszają się po współśrodkowych okręgach ze wspólnym środkiem obrotu ICR.

W modelu „rowerowym” zastępuje pary kół kołem ekwiwalentnym. Z przodu i z tyłu umieszczam po jednym „kole zastępczym” w środku osi, a skręt opisuję pojedynczym kątem $\delta$. Ponieważ w modelu 2WS łuk ruchu wyznacza oś tylna, stan pojazdu definiuję w środku osi tylnej i opisuje zmiennymi stanu $[x, y, \Theta]$. Traktuje je jako współrzędne położenia i orientacji pojazdu w globalnym układzie odniesienia. Promień wycinka łuku kołowego do środka  ICR oznaczam $R_{\mathrm{ICR}}$.

Ten model pozwoli mi pokazać, że sterowanie pojazdem mogę realizować w lokalnym układzie odniesienia za pomocą pary zmiennych $[v, \delta]$ (prędkość i kąt skrętu), a jego efektem jest zmiana położenia i kursu w układzie globalnym, opisana przez zmienne $[x, y, \Theta]$ (współrzędne położenia x,y punktu środka osi tylnej w globalnym układzie odniesienia XOY oraz kąt zwrotu osi pojazdu względem osi OX). Na tej bazie w prosty sposób rozszerzę opis ruchu pojazdu do przypadku 4WS.

#### 2) Model 2WS. Minimum wzorów do opisania zależności kinematycznych
Stosuję model 2WS „rowerowy”. Przyjmuję następujące założenia:
- Koła tylne są sztywne (oś tylna toczna, bez skrętu).
- Koła przednie są skrętne; dopuszczamy różne kąty kół wewnętrznego i zewnętrznego, w ten sposób że proste prostopadłe do powierzchni bocznych przecinają się w jednym punkcie — środku obrotu pojazdu ICR.

<img src="{{ 'assets/images/modelowanie/model2WS.png' | relative_url }}" alt="model2WS" style="width:75%; max-width:100%; height:auto;" />

Otrzymuję związek krzywizny z kątem skrętu:

$$
\begin{aligned}
R_{\mathrm{ICR}} &= \dfrac{L}{\tan \delta}
\end{aligned}
$$

Długość przebytej drogi w kroku czasu $\Delta T$ w ruchu po wycinku łuku kołowego wynosi:

$$
\begin{aligned}
\lambda &= v\,\Delta T \;=\; R_{\mathrm{ICR}}\,\Delta \theta
\end{aligned}
$$

Oznaczenia:

$L$ — rozstaw osi [mm]  
$D$ — rozstaw kół (szerokość toru) [mm]  
$R_w$ — promień koła [mm]  
$\delta$ — kąt skrętu w modelu „rowerowym” (ekwiwalent dla osi przedniej) [rad]  
$R_{\mathrm{ICR}}$ — promień do chwilowego środka obrotu (ICR) mierzony od środka osi tylnej [mm]  
$v$ — prędkość liniowa środka osi tylnej [mm/s]  
$\lambda$ — długość przebytego łuku w kroku $\Delta T$ [mm]  
$\Delta \theta$ — przyrost orientacji w kroku $\Delta T$ [rad]  
$[x, y, \Theta]$ — zmienne stanu: współrzędne i orientacja w układzie globalnym

Równania ruchu (postać ciągła dla punktu środka osi tylnej, układ globalny) przyjmują postać:

$$
\begin{aligned}
\dfrac{dx}{dt} &= v \cos \Theta
\end{aligned}
$$

$$
\begin{aligned}
\dfrac{dy}{dt} &= v \sin \Theta
\end{aligned}
$$

$$
\begin{aligned}
\dfrac{d\Theta}{dt} &= \dfrac{v}{L}\,\tan \delta
\end{aligned}
$$

gdzie oznaczono:

$x, y$ — położenie środka osi tylnej w układzie globalnym  
$\Theta$ — orientacja pojazdu w układzie globalnym  
$v$ — prędkość liniowa wzdłuż osi pojazdu (lokalnie)  
$\delta$ — kąt skrętu w modelu „rowerowym” (ekwiwalent dla osi przedniej)  
$L$ — rozstaw osi

#### 3) Model geometrii samochodu 4WS — skręt przeciwfazowy
Pojazd obraca się wokół punktu przecięcia promieni skrętu obu osi (Instantaneous Center of Rotation, ICR). Dla skrętu przeciwfazowego (kąty skrętu pary kół przód i tył o tej samej wartości, lecz przeciwnie skierowanej) ICR leży na osi symetrii pojazdu, a promień toru środka pojazdu wynosi:

$$
\begin{aligned}
R_{\mathrm{ICR}} &= \dfrac{L}{\tan \delta - \tan(-\delta)} \;=\; \dfrac{L}{2\,\tan \delta}
\end{aligned}
$$

Długość przebytej drogi w kroku czasu $\Delta T$ w ruchu po wycinku łuku kołowego wynosi:

$$
\begin{aligned}
\lambda &= v\,\Delta T \;=\; R_{\mathrm{ICR}}\,\Delta \theta
\end{aligned}
$$

<img src="{{ 'assets/images/modelowanie/model4WS.png' | relative_url }}" alt="model4WS" style="width:50%; max-width:100%; height:auto;" />

Równania ruchu (postać ciągła dla środka pojazdu, układ globalny) przyjmują postać:

$$
\begin{aligned}
\dfrac{dx}{dt} &= v \cos \Theta
\end{aligned}
$$

$$
\begin{aligned}
\dfrac{dy}{dt} &= v \sin \Theta
\end{aligned}
$$

$$
\begin{aligned}
\dfrac{d\Theta}{dt} &= \dfrac{2v}{L}\,\tan \delta
\end{aligned}
$$

Przyjmując ruch bez poślizgu (uproszczenie konieczne dla zachowania „czystej” kinematyki), promienie toru kół (względem tego samego ICR) określam następująco:

- wewnętrzny:  

$$
\begin{aligned}
R_{\mathrm{in}}(\delta) &= \sqrt{\left(R_{\mathrm{ICR}}(\delta) - \dfrac{D}{2}\right)^{2} + \left(\dfrac{L}{2}\right)^{2}}
\end{aligned}
$$

- zewnętrzny:  

$$
\begin{aligned}
R_{\mathrm{out}}(\delta) &= \sqrt{\left(R_{\mathrm{ICR}}(\delta) + \dfrac{D}{2}\right)^{2} + \left(\dfrac{L}{2}\right)^{2}}
\end{aligned}
$$

Dla sterowania „prędkością centralną” ($n_c \propto v$) otrzymuję dyferencjał prędkościowy (prędkości kół dobrane do promieni toru, aby unikać poślizgu):

$$
\begin{aligned}
n_{\mathrm{in}} &= n_c \dfrac{R_{\mathrm{in}}}{R_c}, \quad
n_{\mathrm{out}} &= n_c \dfrac{R_{\mathrm{out}}}{R_c}
\end{aligned}
$$

gdzie $R_c$ to promień toru środka pojazdu (tu $R_c = R_{\mathrm{ICR}}$), $n_c$ to prędkość obrotowa kóła zastępczego w punkcie środka pojazdu  wynikająca z prędkości v lub po prostu zadana prędkość obrotowa środka pojazdu.

#### 4) Model 4WS — pary kół identycznie skrętne, różnica względem Ackermanna
W praktycznej realizacji mojego pojazdu przyjmuję równoległe ustawienie kół po lewej i prawej stronie tej samej osi (brak geometrii Ackermanna na osi).

<img src="{{ 'assets/images/modelowanie/Acker_vs_4WS.png' | relative_url }}" alt="Acker_vs_4WS" style="width:75%; max-width:100%; height:auto;" />

To oznacza: kąty skrętu kół lewe/prawe na danej osi są identyczne, proste prostopadłe do płaszczyzn kół na tej osi są równoległe i nie przecinają się w jednym punkcie, nie istnieje jeden wspólny ICR dla wszystkich kół.

Na ukazanym rysunku koła ustawione są równolegle. Pojazd ma dwa “fikcyjne” ICR (chociaż w rzeczywistości może mieć tylko jeden) narzucane przez pary kół wewnętrznych i zewnętrznych. W konsekwencji pojawiają się poślizgi wiertne (lateral scrub) i dodatkowe momenty; rzeczywiste położenie „efektywnego” ICR oraz rozkład prędkości/poślizgów wynikają z sił kontaktowych opona–podłoże. Aby to ująć, potrzebny byłby model dynamiki opon (np. Pacejka/Magic Formula) lub przynajmniej quasi‑statyczny model sił bocznych.

Stosują w moim modelu zależności kinematyki Ackermanna zakładam idealną symetrię pojazdu i jeden wspólny ICR w punkcie wynikającym z idealnej geometrii (zaznaczonym na rysunku). Wtedy wszystkie płaszczyzny kół przecinają się w tym samym punkcie, co oznacza idealną zbieżność w ujęciu kinematycznym.
W praktyce (po pierwszych przejazdach na macie, których wyniki opiszę później) można zauważyć efekt “przyduszania prędkości” kół zewnętrznych w skręcie. Wynika to prawdopodobnie z wyższej przyczepności podłoża maty, która ogranicza poślizg kompensujący nieidealną zbieżność. W związku z tym w sterowaniu dyferencjałem planuję zmniejszyć różnicę prędkości po stronach poprzez współczynnik redukcji:

$$
\begin{aligned}
n_{\mathrm{in}} &= n_c (1+\kappa) \dfrac{R_{\mathrm{in}}}{R_c}, \quad
n_{\mathrm{out}} &= n_c (1-\kappa) \dfrac{R_{\mathrm{out}}}{R_c}
\end{aligned}
$$

gdzie $n_c$ to prędkość referencyjna (np. prędkość “środkowa”), $R_c$ — promień toru środka pojazdu, a $\kappa \in [0,1)$ jest regulowanym współczynnikiem redukcji różnicy prędkości. Wartość $\kappa$ wyznaczę eksperymentalnie na podstawie przejazdów po okręgu.

#### 5) Obserwacja zmiennych stanu — problem do rozwiązania
Jak pokazałem wcześniej, ruch pojazdu opisany zmiennymi stanu $[x, y, \Theta]$ traktuję jako konsekwencję sterowania parami $[v, \delta]$. Teraz zaglądam „pod maskę” — chcę zobaczyć, co dzieje się, gdy pojawią się błędy i zakłócenia oraz jak wpływają one na proces sterowania (którego celem jest osiągnięcie pożądanego zachowania obiektu mimo zakłóceń).

<img src="{{ 'assets/images/modelowanie/Schemat_sterowania.png' | relative_url }}" alt="Schemat_sterowania" style="width:125%; max-width:100%; height:auto;" />

Rysunek powyżej pokazuje, że szumy i zakłócenia (według miejsca powstawania w łańcuchu sterowania) można rozróżnić następująco:
- sterowania — dodają się do sygnałów sterujących (gaz/hamulec/skręt),
- procesowe — wpływają na sam obiekt (np. efektywny moment napędowy, kąt skrętu),
- pomiarowe — zanieczyszczają wynik pomiaru z czujników.

W moim układzie mogę bezpośrednio nastawiać i mierzyć prędkości obrotowe kół oraz kąty skrętu serw. Natomiast zmienne stanu opisujące ruch (położenie i kurs) nie są mierzalne wprost. Jeśli na nich mi zależy — a to jest właśnie ten przypadek — muszę je wyznaczać pośrednio na podstawie równań modelu. Problemem są tu zakłócenia i szumy. Tę funkcję przejmie obserwator zmiennych stanu.

<img src="{{ 'assets/images/modelowanie/obserwator.png' | relative_url }}" alt="obserwator" style="width:75%; max-width:100%; height:auto;" />

Mój model, w którym umieszczam obserwator, ma dwie warstwy zmiennych: widzialną i ukrytą.

Warstwa widzialna:
- sterowania zadane $u$ (gaz/hamulec/skręt), obarczone szumem sterowania,
- pomiary $y$ (z czujników), obarczone szumem pomiarowym.

Warstwa ukryta:
- rzeczywisty stan $x$ (położenie, orientacja/kurs, prędkości, ewentualne poślizgi), podlegający niepewnościom: realizacyjnym, procesowym i pomiarowym.

<img src="{{ 'assets/images/modelowanie/warstwy_modelu.png' | relative_url }}" alt="warstwy_modelu" style="width:75%; max-width:100%; height:auto;" />

Cel modelowania formułuję następująco: wyznaczyć najlepsze oszacowanie $\hat{x}$ na podstawie $u$ i $y$, mimo błędów modelu i szumów. Jest to problem probabilistyczny; do jego rozwiązania zastosuję prawo propagacji niepewności i filtrację.

Ujęcie równań modelu:

- dynamika:

$$
\begin{aligned}
x_{k+1} &= f(x_k, u_k, d_k) + w_k
\end{aligned}
$$

— szum procesu $w_k$,

- pomiar:

$$
\begin{aligned}
y_k &= h(x_k) + v_k
\end{aligned}
$$

— szum pomiaru $v_k$.

#### 6) Prosta odometria modelu 4WS
Zanim zacznę modelowanie, ustalam sposób sterowania i efekt ruchu oraz wybór modelu. Pojazdem steruję, nastawiając prędkości silników i ustawiając kąty skrętu kół. Efektem jest ruch postępowy albo obrót po łuku wokół chwilowego środka obrotu ICR. Do opisu wystarczy najprostsza kinematyka — model Ackermanna w wariancie 4WS (przeciwfazowo), bez wchodzenia w dynamikę i poślizgi.

Stan pojazdu opisuję wektorem $x = [x, y, \Theta]^{\mathsf T}$, gdzie $x$ i $y$ to współrzędne w globalnym układzie odniesienia, a $\Theta$ to orientacja (kąt zwrotu) nadwozia względem osi OX. Sterowanie zbieram w wektorze $u = [v_{\mathrm{icr}}, \delta]^{\mathsf T}$. W praktyce wygodniej jest sterować prędkością liniową pojazdu $v$ (w środku pojazdu) i kątem skrętu $\delta$, ale w odometrii 4WS można myśleć równoważnie o prędkości „centralnej” związanej z ruchem po okręgu wokół ICR.

Równania aktualizacji stanu otrzymuję przez dyskretyzację równań ruchu. Stosuję prostą dyskretyzację explicite (Euler w przód) z kątem liczonym z poprzedniego kroku, bo w typowym sterowaniu wartości utrzymane są stałe przez cały krok czasu $\Delta T$. Wtedy:

$$
\begin{aligned}
x_k &= x_{k-1} + V_k\,\Delta T \,\cos \Theta_{k-1}
\end{aligned}
$$

$$
\begin{aligned}
y_k &= y_{k-1} + V_k\,\Delta T \,\sin \Theta_{k-1}
\end{aligned}
$$

$$
\begin{aligned}
\Theta_k &= \Theta_{k-1} + \dfrac{2 V_k\,\Delta T}{L}\,\tan \delta_k
\end{aligned}
$$

gdzie: $\delta_k$ to zastępczy kąt skrętu w modelu 4WS przeciwfazowego, $\Theta_k$ to orientacja pojazdu w układzie globalnym, $V_k$ — prędkość liniowa (w środku pojazdu), $L$ — rozstaw osi, a $\Delta T$ — krok czasu. Ten schemat stanowi bazę do odometrii: integruję przebyte odcinki i przyrosty orientacji, korzystając z próbkowanych sygnałów sterujących i/lub z estymowanych prędkości. W praktycznej implementacji ograniczam kąt skrętu do dopuszczalnego zakresu, pilnuję wspólnego zegara dla wszystkich sygnałów oraz — gdy to potrzebne — stosuję korekty (np. filtrację) w celu redukcji dryftu wynikającego z szumów i błędów modelu.

#### 7) Co dalej
W następnej sekcji opiszę wyniki pierwszych jazd i poddam je krytycznej analizie.