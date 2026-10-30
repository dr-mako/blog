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

*Rys. 1. Różnica między map fittingiem a SLAM-em. W pierwszym przypadku trajektoria odometryczna pozostaje ustalona, natomiast w drugim optymalizowane są zarówno pozy pojazdu, jak i położenia landmarków.*


## 2) Ten sam marker w kilku miejscach mapy
W poprzednim eksperymencie pokazałem, że kolejne obserwacje tego samego nieruchomego markera ArUco, po przeliczeniu do globalnego układu współrzędnych, nie zawsze trafiają w jedno miejsce. Przy zamrożonej trajektorii odometrycznej mogłem jedynie wyznaczyć średnią pozycję landmarku i zmierzyć rozrzut jego obserwacji.
W SLAM-ie tę samą informację wykorzystuję inaczej. Położenie markera w świecie jest wspólne dla wszystkich jego obserwacji, natomiast pozycje i orientacje pojazdu mogą się zmieniać podczas optymalizacji.
Jeżeli samochód obserwuje marker z różnych miejsc, każda obserwacja nakłada ograniczenie na wzajemne położenie pojazdu i landmarku. Algorytm musi więc znaleźć takie pozycje pojazdu i markerów, które możliwie dobrze uzgodnią te pomiary, nie odchodząc przy tym nadmiernie od ruchu zarejestrowanego przez odometrię.
Szczególnie przydatne są ponowne obserwacje tych samych markerów po przejechaniu pętli. Łączą one fragmenty trajektorii, które są odległe w czasie, ale odnoszą się do tych samych nieruchomych punktów. Dzięki temu mogą ograniczać narastającą niespójność położenia.
Nie oznacza to jeszcze, że SLAM odtworzy rzeczywistą trajektorię. Wynik zależy od jakości pomiarów oraz przyjętych ograniczeń. Możemy jednak sprawdzić, czy wspólna optymalizacja pozwala uzyskać bardziej spójną trajektorię i mapę niż przy zamrożonej odometrii.

## 3) Minimalna teoria

W chwili $\(k\)$ pojazd ma pozę w układzie globalnym:

$$
\[
\mathbf{x}_k=
\begin{bmatrix}
x_k & y_k & \varphi_k
\end{bmatrix}^{T}
\]
$$

Z poprzedniego etapu przetwarzania obrazu otrzymuję lokalną obserwację markera $\(j\)%, wyrażoną w metrach względem pojazdu:

$$
\[
\mathbf{z}_{k,j}^{\mathrm{local}}=
\begin{bmatrix}
Y_{\mathrm{BEV}}\\
-X_{\mathrm{BEV}}
\end{bmatrix}
\]
$$

Geometria kamery względem pojazdu została już uwzględniona podczas przygotowania BEV. W SLAM-ie nie optymalizuję dodatkowych parametrów tej transformacji.
Znając pozę pojazdu, mogę przeliczyć obserwację markera do układu globalnego:

$$
\[
\hat{\mathbf{p}}_{k,j}^{G}=
\begin{bmatrix}
x_k\\
y_k
\end{bmatrix}
+
R(\varphi_k)\mathbf{z}_{k,j}^{\mathrm{local}}
\]
$$

gdzie $\(R(\varphi_k)\)$ jest macierzą obrotu wynikającą z orientacji pojazdu.

W mapie marker o identyfikatorze $\(j\)$ ma jedną estymowaną pozycję $\(\mathbf{p}_j^G\)$. Różnicę pomiędzy pozycją wynikającą z obserwacji a pozycją w mapie opisuję residualem:

$$
\[
\mathbf{r}_{\mathrm{obs},k,j}
=
\hat{\mathbf{p}}_{k,j}^{G}-\mathbf{p}_j^G
\]
$$

W aktualnym programie residual obserwacji obliczam równoważnie w lokalnym układzie pojazdu. Porównuję zmierzone położenie markera z położeniem przewidywanym na podstawie aktualnej pozy pojazdu i mapy:

$$
\[
\mathbf{r}_{\mathrm{obs},k,j}^{\mathrm{local}}
=
R(\varphi_k)^T
\left(
\mathbf{p}_j^G-
\begin{bmatrix}x_k\\y_k\end{bmatrix}
\right)
-
\mathbf{z}_{k,j}^{\mathrm{local}}
\]
$$

Obie postacie mają tę samą długość residualu, ponieważ różnią się jedynie obrotem.
Drugim rodzajem ograniczenia jest odometria. Dla kolejnych węzłów trajektorii znam względny ruch wynikający z odometrii, oznaczony jako $\(\Delta\mathbf{x}_k^{\mathrm{odom}}\)$. Z aktualnie estymowanych poz mogę obliczyć analogiczny ruch $\(\Delta\mathbf{x}_k^{\mathrm{SLAM}}\)$.

Residual odometryczny ma zatem postać:

$$
\[
\mathbf{r}_{\mathrm{odom},k}
=
\Delta\mathbf{x}_k^{\mathrm{SLAM}}
-
\Delta\mathbf{x}_k^{\mathrm{odom}}
\]
$$

Różnicę orientacji sprowadzam do przedziału od $\(-\pi\)$ do $\(\pi\)$, aby poprawnie uwzględnić okresowość kąta.
Algorytm szuka takich poz pojazdu i pozycji landmarków, które zmniejszają obie grupy residuali. Nie są one jednak traktowane jednakowo. Residuale normalizuję przez przyjęte niepewności pomiarów: 30 mm dla translacji odometrycznej, 2° dla zmiany orientacji oraz 50 mm dla lokalnych obserwacji landmarków.
Zamiast zwykłej sumy kwadratów wykorzystuję funkcję straty Huber, która ogranicza wpływ dużych residuali. Do funkcji celu dodaję również ograniczenie pierwszej pozy pojazdu, czyli anchor, aby ustalić układ odniesienia.
W ten sposób SLAM szuka kompromisu między ruchem wynikającym z odometrii a położeniami nieruchomych markerów. W przeciwieństwie do map fittingu może przy tym zmieniać całą trajektorię.

### 4) Jak to działa intuicyjnie, czyli dwie sprzeczne „prawdy”
Najprościej myśleć o tym jak o kompromisie między dwiema informacjami:

-	Odometria mówi: „między klatką $k$ i $k+1$ przesunąłem się mniej więcej tak”.
-	Landmarki mówią: „marker o ID=7 jest jeden i stoi w świecie w jednym miejscu, więc jeśli widzę go ponownie, moja trajektoria musi się z tym zgodzić”.

SLAM offline bierze cały log przejazdu i próbuje znaleźć takie pozycje pojazdu w czasie oraz takie położenia markerów, żeby te dwie „prawdy” dało się pogodzić możliwie najlepiej. U mnie dodatkowo używam funkcji odpornej na outliery, żeby pojedyncze złe detekcje nie obciążały wyniku.
Aby zapewnić możliwość porównywania wyników różnych metod obliczeniowych między sobą narzucam punkt odniesienia w miejscu pierwszego zarejestrowanego markera. W moim przypadku przejazd zaczynam od markera id=1 więc przyjmuję, że stanowi on początek globalnego „światowego” układu współrzędnych. 


### 5) MEtap kalibracyjny: gdy SLAM uczy się błędów systematycznych
Zanim zacznę traktować wynik jak „lokalizację”, pozwalam SLAM-owi znaleźć również kilka prostych korekt systematycznych. To ważne, bo w praktyce większość problemów nie jest czystym szumem Gaussa, tylko konsekwencją drobnych, ale stałych błędów:

-	skala px $\to$ m w BEV może być minimalnie zła,
-	odometria może zaniżać/zawyżać narastanie kąta,
-	w zakrętach może pojawić się systematyczny dryf boczny (poślizg, nieliniowość modelu skrętu, sprężystość opon itd.).

Na danych z przejazdu optymalizacja znalazła m.in. efektywną skalę około $0.000950$ m/px (około 5% różnicy względem nominalnego $0.001$ m/px) oraz niewielkie korekty parametrów związanych z yaw/krzywizną, plus prosty parametr bocznego „uciekania”. Najważniejszy fakt nie jest jednak w samych liczbach, tylko w konsekwencji: żeby złożyć spójny świat, trajektoria musiała się realnie przesunąć względem odometrii nawet o dziesiątki centymetrów. Tego map fitting nie mógł zrobić z definicji.
Po tym etapie moja sytuacja wygląda jak na poniższym rys. Otrzymane parametry korekt zapisuję i traktuję jako stałe.

<img src="{{ 'assets/images/SLAMoffline/SlamCalib.png' | relative_url }}" alt="SlamCalib" style="width:100%; max-width:100%; height:auto;" />

### 6) Etap „runtime” offline: biasy zamrożone, pracuje tylko mapa i trajektoria
W drugim kroku uruchamiam SLAM jeszcze raz, ale już bez „uczenia się” powyższych korekt. One są zamrożone, a optymalizacja poprawia tylko:
trajektorię w czasie $\leftrightarrow$ mapę markerów.
W praktyce wynik nadal wymagał korekty trajektorii względem surowej odometrii (średnio około 0.38 m, maksymalnie około 0.82 m). To jest spójne z intuicją z poprzedniego wpisu: problem najbardziej wychodzi w manewrach wymagających (zakręty), gdzie uproszczony model ruchu zaczyna rozmijać się z fizyką. Moje wyniki ukazują rys. poniżej. Porównanie wyznaczonych pozycji landmarków z ich prawdziwymi (nie znanymi przez algorytm) pozycjami Ground Truth nie wygląda imponująco. W stosunku do GT landmarki mapy wykazują następujące błędy położenia  RMSE [m]: 0.355, Mean [m]: 0.317, Max  [m]: 0.568. 

<img src="{{ 'assets/images/SLAMoffline/SlamRuntime.png' | relative_url }}" alt="SlamRuntime" style="width:100%; max-width:100%; height:auto;" />

### 7) Porównanie z Ground Truth: sztywne wyrównanie nie wystarcza, przekształcenie afiniczne ujawnia prawdę
Ponieważ układ markerów na makiecie mam zmierzony, mogłem porównać mapę uzyskaną ze SLAM z Ground Truth. I tu pojawia się rzecz, która na pierwszy rzut oka wygląda jak porażka, a w praktyce jest świetnym testem diagnostycznym: jakiego typu błędy dominu¬ją w całym pipeline’ie.
Najpierw zrobiłem najprostsze możliwe wyrównanie: potraktowałem wynik SLAM jak „sztywny obiekt” i dopasowałem go do GT tylko przez przesunięcie i obrót. Gdyby problem polegał wyłącznie na tym, że SLAM ma inny punkt zerowy albo jest obrócony w inną stronę, to takie wyrównanie powinno niemal zlikwidować różnice. Tymczasem po tym kroku błąd nadal był rzędu dziesiątek centymetrów (RMSE około 0.36 m). To sugeruje, że nie chodzi tylko o wybór układu współrzędnych — w wyniku jest coś, czego nie da się skorygować samym „przesuń i obróć”.
Potem dopuściłem wyrównanie afiniczne (rys.). To nadal jest proste dopasowanie globalne, ale daje dodatkową swobodę: poza przesunięciem i obrotem pozwala na inną skalę w osiach oraz lekki „skos” (czyli sytuację, w której osie w praktyce mieszają się ze sobą). Ważne jest jednak, co tak naprawdę robi tu „affine”: to nie jest kolejny etap SLAM-u, tylko czysta procedura dopasowania geometrii. Mamy dwie chmury punktów — landmarki ze SLAM oraz ich pozycje z Ground Truth — i zakładamy, że GT jest znane. Następnie szukamy takiej transformacji afinicznej, która możliwie najlepiej nałoży jedną chmurę na drugą (w sensie minimalizacji średniego błędu). Dopiero po takim dopasowaniu błąd spada do około 6.7 cm RMSE. Jeden marker nadal odstaje wyraźniej (około 15 cm), ale cała struktura układu robi się zaskakująco zgodna.

<img src="{{ 'assets/images/SLAMoffline/SlamAffine.png' | relative_url }}" alt="SlamAffine" style="width:100%; max-width:100%; height:auto;" />

To prowadzi do mocnego, ale jednocześnie ostrożnego wniosku:
„After affine alignment the landmark RMSE equals 6.7 cm, indicating high structural consistency of the SLAM reconstruction. The residual deformation suggests systematic bias in BEV metric scaling and/or vehicle kinematic model.”
Po polsku: SLAM zrekonstruował układ markerów spójnie, tylko że ta spójność jest opisana w geometrii, która nie jest idealnie zgodna z „metryką z miarki”. To może oznaczać systematyczne odkształcenie po stronie obserwacji (np. BEV nie jest idealnie metryczny), ale nie byłbym tu kategoryczny. Największa pozostała niezgodność dotyczy markera 8, a ten leży dokładnie tam, gdzie przejazd zawierał ostry zakręt. W takich miejscach uproszczony model Ackermanna najczęściej zaczyna przegrywać z rzeczywistością: promień skrętu nie zgadza się z modelem, pojawia się poślizg boczny, a tor jazdy „płynie”. Tego nie da się w pełni zamknąć jedną stałą korektą.
Dlatego na tym etapie najuczciwszy opis brzmi tak: rekonstrukcja SLAM jest strukturalnie spójna, ale resztkowa deformacja wskazuje na obecność błędów systematycznych — prawdopodobnie jednocześnie po stronie geometrii obserwacji (BEV) i po stronie modelu ruchu (szczególnie w ostrych zakrętach).

### 8) Co to zmienia w interpretacji „czy SLAM działa”?
Na tym etapie najważniejsza zmiana myślenia jest taka:
•	SLAM nie polega na tym, że „koszt spada i wykres wygląda ładnie”.
•	SLAM polega na tym, że potrafi ujawnić, gdzie świat przestaje pasować do modelu.
U mnie wyszło jasno: SLAM domknął spójność landmarków i skorygował trajektorię, ale porównanie z GT pokazało, że resztka błędu ma charakter geometrycznej deformacji zgodnej z affine. To nie jest „błąd optymalizacji”. To jest informacja o modelu obserwacji (BEV) i/lub o uproszczeniach ruchu.

### 9) Co dalej: dwa kierunki, które mają sens
Na tym etapie SLAM wymusił spójność między trajektorią a landmarkami i pokazał, gdzie kończą się możliwości „wiary w odometrię”. Jednocześnie porównanie z Ground Truth ujawniło, że resztkowy błąd nie wygląda jak przypadkowy szum, tylko jak błąd systematyczny. I tu pojawia się najważniejsze pytanie na dalszą część projektu: czy dominującym źródłem tej resztki jest geometria obserwacji (metryczność BEV), czy dynamika ruchu (poślizg i odejście od idealnego Ackermanna w ostrych zakrętach), a najpewniej — jaka jest proporcja jednego i drugiego.
Dalsza praca naturalnie dzieli się więc na dwa kierunki. Pierwszy to dopracowanie modelu pomiaru: zamiast zakładać, że BEV jest idealnie metryczny, trzeba sprawić, żeby faktycznie taki był (albo jawnie dopuścić w modelu prostą korektę, która kompensuje zniekształcenie). Drugi to dopracowanie modelu ruchu: skoro największy błąd pojawia się w miejscach o dużej krzywiźnie toru, to warto opisać poślizg jako zjawisko zależne od manewru, a nie jako stałą poprawkę. W praktyce można to zrobić bardzo „inżyniersko”: potraktować trajektorię ze SLAM (wyrównaną do GT) jako odniesienie i na jej podstawie wyznaczyć, jak odometria systematycznie myli się w funkcji skrętu i prędkości, a potem tę korektę włączyć z powrotem do odometrii. To jest moment, w którym SLAM przestaje być tylko algorytmem mapowania — staje się narzędziem do tego, żeby z danych wyciągnąć poprawki do modelu, który dotąd był tylko założeniem.


