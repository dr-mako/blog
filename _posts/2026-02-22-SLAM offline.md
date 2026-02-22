---
layout: post
title: "SLAM offline"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Kiedy trajektoria wreszcie może się poprawić”?<!--more-->


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
W poprzednim wpisie („map fitting”) celowo „zamroziłem” trajektorię i potraktowałem ją jako prawdę. W efekcie pozwala to zobaczyć, że jeśli odometria dryfuje, to mapa musi się zdeformować, ponieważ algorytm jej nie poprawia. Teraz robię krok dalej. Wchodzimy w SLAM — etap, w którym trajektoria nie jest traktowana jako parametr stały.
SLAM (Simultaneous Localization And Mapping) to metoda równoczesnego budowania mapy otoczenia i lokalizowania się na niej na podstawie obserwacji. Klasycznie robi się to „bez mapy wejściowej”: mapa dopiero powstaje. Z drugiej strony możemy ją też stosować do "lokalizacji" na gotowej mapie lub do kalibracji modelu ruchu. Obecny wpis dotyczy wersji pośredniej, ale bardzo praktycznej: SLAM offline, czyli postprocessing na podstawie zapisanych logów przejazdu. Nie ma presji czasu rzeczywistego, jest za to geometria, optymalizacja i możliwość zadania sobie trudnych pytań: co tak naprawdę jest niezgodne — odometria, kamera, a może oba naraz?


### 2) Problem: ten sam marker w kilku miejscach mapy
Eksperyment jest prosty: na podłodze leżą markery ArUco. Każdy ma unikalne ID, więc asocjacjia danych jest ułatwiona: „marker 7” to zawsze ten sam obiekt w świecie.
W każdej klatce obrazu kamera widzi markery, a po przekształceniu do BEV (bird’s-eye view) potrafię odczytać ich położenia w pikselach. To kuszące, żeby traktować BEV jak „metryczną mapkę”, tylko że w praktyce łańcuch przekształceń zawsze ma błędy: trochę w skali, trochę w orientacji, trochę w modelu ruchu. 
Ta sytuację ilustruje poniższy rys., gdzie pokazano obrazy z kamery nałożone na mapę trajektorii odometrycznej (co odpowiada założeniom „map fitting”) w pobliżu ostrego zakrętu z widocznym markerem nr. 8. Te dwa obrazy były zarejestrowane w odstępie 30 klatek. Widoczne jest istotne przesunięcie pozy markera, chociaż oczekiwałbym, że będą w tym samym miejscu mapy. Co jest tu jeszcze bardzo istotne z punktu widzenia SLAM – pokazany przejazd zawiera dwie pętle. To warunek konieczny aby SLAM mógł prawidłowo uzgodnić położenie trajektorii i markerów na jednej mapie. 
 

<img src="{{ 'assets/images/SLAMoffline/marker.png' | relative_url }}" alt="marker" style="width:100%; max-width:100%; height:auto;" />

W map fittingu mogłem „ratować sytuację” deformując mapę. W SLAM-ie celem jest coś innego: zbudować mapę i trajektorię, które są ze sobą spójne

### 3) Minimalna teoria 
W chwili $k$ pojazd ma pozę 
$$x_k=(x_k,y_k,\phi_k)$$
w globalnym układzie świata (WORLD). Detektor zwraca marker (landmark) o ID $j$ w obrazie BEV jako punkt w pikselach 
$$(x_{\text{pix}},y_{\text{pix}})$$ 
Dalej zachodzi prosty łańcuch przekształceń: piksele $\to$ metry w BEV $\to$ stała korekta o ekstrynsykę kamery/BEV względem pojazdu (przesunięcie i stały offset yaw) $\to$ punkt względny w układzie pojazdu $\to$ projekcja do WORLD z użyciem aktualnej pozy pojazdu.
Jeżeli chcę to zapisać jednym zdaniem matematycznym (tylko po to, żeby nie zgubić sensu), to „pozycja w świecie wynikająca z pomiaru” ma postać:

$$\hat{p}_{j,k}= \begin{bmatrix}x_k\\y_k\end{bmatrix} +R(\phi_k)\,z_{\text{veh},j,k}.$$


W tym zapisie 
$$z_{\text{veh},j,k}\in\mathbb{R}^2$$
oznacza obserwację markera w układzie pojazdu (po przeliczeniu BEV na metry i po uwzględnieniu ekstrynsyki), a $R(\phi_k)$ jest macierzą obrotu 2D wynikającą z orientacji pojazdu $\phi_k$.
Tymczasem $p_j\in\mathbb{R}^2$ to bieżąca estymata mapowej pozycji tego samego markera w WORLD. Sedno SLAM-u jest proste: chcę, żeby to, co wynika z pomiarów, zgadzało się z tym, co jest w mapie. Dlatego błąd dopasowania (residual) definiuję jako:

$$r_{\text{obs},k,j}=\hat{p}_{j,k}-p_j.$$

To jednak wciąż nie wszystko, bo trajektoria też nie jest stała. Odometria narzuca dodatkowe ograniczenie: kolejne pozy pojazdu powinny być zgodne z tym, co zmierzył model ruchu. Tę niezgodność opisuję residualem $r_{\text{odom},k}$ między klatkami $k$ i $k+1$. W efekcie algorytm szuka kompromisu minimalizującego łączny błąd:

$$\min_{\{x_k\},\{p_j\}} \sum_k \left\| r_{\text{odom},k} \right\|^2 + \sum_{(k,j)\in\mathcal{O}} \left\| r_{\text{obs},k,j} \right\|^2,$$

gdzie $\mathcal{O}$ jest zbiorem wszystkich zaobserwowanych par „klatka–marker”. W map fittingu mogłem zmieniać tylko $p_j$ (mapę) przy zamrożonej trajektorii. W SLAM-ie zmieniają się jednocześnie $p_j$ oraz $x_k$ (trajektoria) — i to jest ta jedna różnica, która robi całą robotę.


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


