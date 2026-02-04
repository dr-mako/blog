---
layout: post
title: "Wykrywanie pasów ruchu w BEV (bird’s eye view)"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Metoda, kompromisy i przygotowanie pod mapę<!--more-->


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
W poprzednim kroku sprowadziłem obraz z rybiego oka do układu drogi (bird’s eye view, BEV). To była decyzja czysto pragmatyczna: jeśli chcę sterować i składać mapę, potrzebuję geometrii, która zachowuje się możliwie „metrycznie”. W BEV jeden piksel odpowiada w przybliżeniu stałemu odcinkowi na podłodze, a pas przestaje być efektem perspektywy i dystorsji — staje się faktycznym kształtem toru.
Ten wpis jest o następnym etapie: jak z takiego obrazu w BEV wyciągam pas w postaci punktów, które później mogę wykorzystać do sterowania i do składania mapy przejazdu.
To, co opisuję, jest na razie rozwiązaniem „na bogato”: świetnym do przygotowań, strojenia parametrów i masowego przeglądania logów. Na komputerze PC taki pipeline sprawdza się dobrze przy obróbce nagrań, ale w tej formie jest zbyt złożony, żeby Jetson liczył go w czasie jazdy. Dlatego mapę będę składał po przejeździe, a na pokładzie zostanie uproszczona detekcja potrzebna do sterowania.
Warto też doprecyzować, czym w tym projekcie są „pasy ruchu”. Fizycznie to cienkie, białe elementy drukowane 3D (około 2–3 mm grubości) łączone na klipsy jak w składanych torach. Dzięki temu są powtarzalne i mają wyraźną krawędź, ale potrafią też łapać refleksy w trudnym oświetleniu.

Dla człowieka „wygięta” szachownica nie stanowi problemu — mózg koryguje to automatycznie. Dla algorytmu sterowania, który ma wykrywać krawędzie pasa i utrzymywać pojazd w jego środku, jest to już realna przeszkoda. To samo miejsce na torze może wyglądać inaczej w kolejnych klatkach, a proste operacje geometryczne przestają działać stabilnie.
Najlepsze własności geometryczne ma sytuacja, w której obserwujemy podłogę prostopadle z góry — „z lotu ptaka”. Taki obraz ma w przybliżeniu stałą skalę, a linie na podłodze pozostają liniami. Dokładnie tego chcę. Dlatego wprowadzam korektę obrazów do widoku z lotu ptaka (bird’s eye view, BEV). Zakładam przy tym, że interesuje mnie przede wszystkim płaszczyzna drogi (mata/tor), a wszystko, co jest „pionowe”, traktuję jako efekt uboczny. BEV nie jest rekonstrukcją 3D — to świadome uproszczenie: buduję przekształcenie 2D, które na podłodze daje obraz o bardziej stałej geometrii, wygodny do sterowania, składania mapy z wielu klatek oraz ewentualnego uczenia sieci.
Przekształcenie (fisheye $\rightarrow$ bird’s eye / usunięcie dystorsji) musi się jednak „z czegoś wziąć” — potrzebuję informacji, jak piksele obrazu odpowiadają punktom na podłodze. Są dwa sensowne sposoby, żeby tę zależność wyznaczyć.


### 2) Segmentacja: „znajdź pas”, ale nie daj się oszukać oświetleniu
Na pierwszy rzut oka zadanie wydaje się proste: pas jest jasny, tło jest ciemniejsze. W praktyce problemem nie jest sam pas, tylko to, że jasność tła nie jest stała. Nawet na dość jednorodnej macie pojawiają się gradienty od okna, cienie, delikatne zmiany ekspozycji, a czasem lokalne refleksy. Jeśli zastosuję zwykłe progowanie jasności, to w jednym miejscu pas wyjdzie idealnie, a w innym zacznie się „wylewać” całe tło. Schemat kroków (BEV → ROI → tło → Otsu → morfologia → punkty) ukazany jest na rysunku:

<img src="{{ 'assets/images/WykrywaniePasow/lane pipeline.png' | relative_url }}" alt="lane pipeline" style="width:75%; max-width:100%; height:auto;" />

Dlatego pierwszym krokiem po BEV nie jest progowanie, tylko wyrównanie tła. Intuicja jest taka: wolnozmienne zmiany jasności traktuję jako „oświetlenie”, a pas jako strukturę, którą chcę z tego oświetlenia wyciągnąć. Technicznie da się to zrobić na różne sposoby, ale sens jest zawsze ten sam: oszacować łagodne tło i je odjąć. Po takim zabiegu pas zostaje wyraźny, a podłoga przestaje udawać obiekt.
Dopiero po wyrównaniu tła robię lekkie wygładzenie (żeby uspokoić fakturę maty), a potem globalne progowanie (np. metodą Otsu, z ewentualną drobną regulacją progu). To jest celowo proste: na tym etapie nie próbuję „zrozumieć obrazu”, tylko chcę dostać stabilną binarną maskę, która w większości klatek mówi: tu jest pas.
Wynik po progowaniu prawie zawsze wymaga sprzątania. Wchodzą tu klasyczne narzędzia morfologii, czyli operacje „porządkujące kształty”: domykanie łączy przerwy i zasklepia drobne dziury, a filtr minimalnej powierzchni wyrzuca śmieci. Potem stosuję jeszcze jeden praktyczny trik: zostawiam tylko obiekty odpowiednio „długie”. Pas na torze jest strukturą wydłużoną, więc jeśli coś jest krótką plamą, to zwykle jest artefaktem.
Na typowych klatkach działa to bardzo dobrze. W BEV dostaję czyste krawędzie pasa, a pierwsze dwa przykłady przejazdu (bez blików) pokazują ten „happy path”: segmentacja jest stabilna, a wykryty kształt jest spójny i gładki w kolejnych klatkach.
Te „poprawne” sytuacje pokazuje poniższy rysunek:

<img src="{{ 'assets/images/WykrywaniePasow/frame_0252_1.png' | relative_url }}" alt="frame_0252_1" style="width:75%; max-width:100%; height:auto;" />

— widać na nim kolejne etapy przetwarzania, od surowej klatki do punktów, które mogę dalej wykorzystać w sterowaniu i mapowaniu. W lewym górnym rogu znajduje się obraz wejściowy w skali szarości (jeszcze w geometrii rybiego oka). W prawym górnym rogu jest ten sam fragment sceny po przekształceniu do widoku z lotu ptaka (BEV): podłoga ma już „prawie metryczną” geometrię, a obszary poza mapą/ROI są wycięte (czarne trójkąty).
Lewy dolny panel to wynik segmentacji binarnej \(BW\) po wyrównaniu tła, wygładzeniu, progowaniu oraz sprzątaniu morfologicznym. To jest moment, w którym algorytm mówi wprost: „to są piksele pasa”. W prawym dolnym rogu pokazuję tę samą detekcję w formie punktów obrysu — czyli piksele krawędzi wyciągnięte z maski (obrys). Taka reprezentacja jest wygodna w dalszych krokach: można na niej dopasowywać krzywe, wyznaczać oś pasa i odkładać punkty do wspólnej mapy przejazdu.

### 3) Obrys, szkielet i „punkty, które da się użyć”
Maska binarna jest dobra do podglądu, ale do dalszego przetwarzania wolę punkty. I tu są dwie sensowne reprezentacje, zależnie od tego, co chcę robić dalej:
•	obrys pasa — punkty na granicy wykrytego obiektu,
•	szkielet — jednopikselowa oś obiektu.
Obrys dobrze oddaje geometrię krawędzi, co jest intuicyjne przy pasach ruchu. Szkielet bywa stabilniejszy, gdy segmentacja miejscami robi pas grubszy lub cieńszy, bo sprowadza obiekt do jednej krzywej. Ponieważ BEV jest metryczny, te punkty są od razu punktami w układzie drogi — i to jest dokładnie to, czego potrzebuję do mapowania.
Na poniższym rys. widzimy, że nawet w bardzo zakrzywionej perspektywie możemy prawidłowo wyznaczyć punkty obrysu.

<img src="{{ 'assets/images/WykrywaniePasow/frame_0310_2.png' | relative_url }}" alt="frame_0310_2.png" style="width:75%; max-width:100%; height:auto;" />

### 4) Największy wróg: bliki i niskie słońce
Następny obrazek jest dla mnie najcenniejszy, bo pokazuje realne ograniczenie tej metody. W bardzo słoneczny zimowy dzień, gdy słońce świeci pod małym kątem, potrafią pojawić się rozległe refleksy. Wtedy „jasne” przestaje znaczyć „pas”, tylko „cokolwiek, co akurat złapało blik”. Progowanie dostaje fałszywe, bardzo jasne regiony i maska może rozsypać się w spektakularny sposób.
I tu dochodzę do ważnej decyzji projektowej. Na tym etapie nie próbuję agresywnie walczyć z blikami filtrami „anty-refleks”, bo bardzo łatwo jest przy okazji uszkodzić sam pas. Wolę mieć pipeline, który w trudnych warunkach jawnie zaczyna produkować błędy (i można takie klatki odsiać albo potraktować inaczej), niż pipeline, który pozornie działa, ale cicho gubi poprawne fragmenty.
Co więcej, w tym konkretnym przypadku najlepsze „ulepszenie algorytmu” jest poza algorytmem: po prostu stabilizuję warunki pracy kamery. W praktyce oznacza to osłonięcie okna, żeby wyciąć niskie słońce i ograniczyć refleksy u źródła. Ewentualnie można zastosować oświetlenie drogi z pojazdu (np. LED). Rys. poniżej ukazuje sytuację krytyczną – przebieg pas jest tu prawie niewidoczny.

<img src="{{ 'assets/images/WykrywaniePasow/frame_0624_3.png' | relative_url }}" alt="frame_0624_3.png" style="width:75%; max-width:100%; height:auto;" />

### 5) Sterowanie: środek pasa, a co gdy widać tylko jedną krawędź?
Docelowo najwygodniejsza sytuacja jest wtedy, gdy widzę dwie krawędzie. Wtedy problem prowadzenia upraszcza się do geometrii: z lewej i prawej krawędzi wyznaczam oś pasa i steruję tak, żeby pojazd trzymał się środka.
Wiem jednak, że to nie zawsze będzie możliwe. W zakręcie albo przy ograniczonym polu widzenia może się zdarzyć, że jedna krawędź „znika”. Tego jeszcze nie rozpracowałem w tej iteracji, ale naturalne są dwa kierunki: albo przejść w tryb jazdy „z offsetem” od jednej krawędzi (w BEV to jest proste, bo odległości są metryczne), albo wykorzystać mapę/pamięć trasy, żeby brakujący fragment domyślić z kontekstu.
Ten rys pokazuje taką sytuację – algorytm rozpoznaje tutaj tylko jedną stronę pasa ruch.

<img src="{{ 'assets/images/WykrywaniePasow/frame_0891_4.png' | relative_url }}" alt="frame_0891_4.png" style="width:75%; max-width:100%; height:auto;" />

W mojej wcześniejszej wersji sterowania bardzo dobrze sprawdziło się podejście pościgowe. Zamiast sterować bezpośrednio na krzywiznę, wybierałem „wirtualną kropkę” przed pojazdem i jechałem w jej kierunku — jakby ta kropka ciągnęła pojazd na dyszlu. Taki układ, lekko domknięty prostym regulatorem (PID), działał zaskakująco stabilnie. BEV jest do tego wręcz stworzone: „kropka” ma sensowną geometrię w układzie drogi, a nie w zdeformowanej perspektywie.

### 6) W następnym kroku mapa: mniej SLAM-u, więcej odometrii (i lekka korekta wizyjna)
Poprzednio budowałem mapę metodą DP SLAM bez odometrii, co było obliczeniowo kosztowne. Teraz sytuacja jest inna: mam odometrię, której ufam na tyle, że może być podstawą predykcji ruchu. To zmienia filozofię całego układu:
odometria robi „ciężką robotę” w sensie przesunięć i obrotów między klatkami, a wizja nie musi wymyślać ruchu od zera — może pełnić rolę korekty, szczególnie przy poślizgach.
Naturalna funkcja celu w takim dopasowaniu to zgodność bieżącej obserwacji z tym, co już jest w mapie. W moich wcześniejszych eksperymentach dobrze sprawdzała się korelacja z wagami (środek obrazu ważniejszy niż brzegi), bo to, co jest najbliżej pojazdu i najbardziej „na wprost”, zwykle jest najbardziej informacyjne i najmniej podatne na artefakty.
Wspominałem też o RANSAC, ale tu trzeba być uczciwym: ma sens dopiero wtedy, gdy mam co dopasowywać. Jeśli obraz jest prawie gładki i jedyną strukturą są dwie białe krawędzie, to RANSAC na klasycznych cechach może nie mieć „punktów zaczepienia”. Natomiast da się go użyć sprytniej: nie do dopasowania cech typu ORB/SIFT, tylko do dopasowania modelu do punktów pasa — czyli do dopasowania transformacji między chmurami punktów (bieżąca klatka kontra fragment mapy). To jest kierunek, który warto sprawdzić, bo potencjalnie mógłby korygować poślizgi nawet wtedy, gdy odometria chwilowo kłamie.
I znów: nie obiecuję wykonywania mapy w czasie rzeczywistym na Jetsonie. Bardziej realistyczne jest, że mapowanie będzie procesem offline albo pół-online, a na pokładzie zostanie tylko ta część, która jest potrzebna do stabilnego prowadzenia.
