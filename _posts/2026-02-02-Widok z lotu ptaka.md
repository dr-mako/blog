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

Przekształcenie do widoku z lotu ptaka nie bierze się „znikąd” — musi wynikać z relacji pomiędzy tym, co widzi kamera, a rzeczywistą geometrią podłogi. Innymi słowy: potrzebuję wiedzieć, który piksel obrazu odpowiada któremu miejscu na płaszczyźnie, po której jedzie pojazd.
Pierwsze, najbardziej popularne podejście, to klasyczna kalibracja w stylu OpenCV: wykonuje się serię zdjęć szachownicy z wielu ujęć, a algorytm dopasowuje parametry modelu kamery (ogniskową, punkt główny i dystorsję). Potem, mając model, można „odkrzywić” obraz i obliczyć rzutowanie na podłogę. To rozwiązanie jest wygodne i w dużej mierze automatyczne, ale w przypadku obiektywu typu rybie oko często prowadzi do praktycznego problemu: poprawne odkształcenie wymaga tak silnej rektyfikacji, że użyteczne pole widzenia wyraźnie się kurczy (dużo obrazu trzeba przyciąć lub „wypada” poza ramkę).
Drugie podejście jest mniej standardowe i bardziej „inżynierskie”: zamiast budować parametryczny model optyki, tworzę bezpośrednie mapowanie na podstawie korespondencji punktów. Na podłodze układam siatkę o znanym kroku, a następnie ręcznie klikam na obrazie w narożniki wybranych pól. W efekcie powstaje tabela par $(pix_x, pix_y) \leftrightarrow (square_x, square_y)$, czyli konkretnych odpowiedników piksel–punkt na podłodze. Rys. 3 pokazuje „kliknięte” punkty narożników, a obok fragment tabeli zależności pomiędzy współrzędnymi pikseli i współrzędnymi tego samego miejsca na siatce.
Na tej podstawie buduję mapę przekształcenia (lookup/remap), która dla każdego punktu w układzie podłogi wskazuje, skąd w obrazie źródłowym pobrać wartość. Wadą tej metody jest pracochłonność i konieczność „oczyszczenia” ręcznie zebranych danych, ale jej największą zaletą jest pełna kontrola nad obszarem roboczym: przy odpowiedniej rozdzielczości i wystarczającej liczbie punktów mogę pokryć praktycznie całe użyteczne pole widzenia, również dla rybiego oka. W tym projekcie priorytetem jest właśnie stabilny, metryczny widok podłogi bez sztucznego zawężania sceny — dlatego wybieram drugą metodę.


### 3 Mapy $Xmap$ i $Ymap$ — co to właściwie jest?

Efektem „klikania” nie jest żadna magiczna funkcja, tylko dwie zwykłe macierze liczb: $Xmap$ i $Ymap$. Powstają one przez interpolacyjne „wypełnienie” całego obszaru roboczego na podstawie zebranych korespondencji. Można je rozumieć jak tablicę odsyłaczy: dla każdego piksela obrazu wyjściowego (BEV w układzie drogi) mówią, z jakiego miejsca w obrazie źródłowym należy pobrać próbkę. W skrócie:

$$
\begin{aligned}
I_{out}(u,v) = I_{src}(Xmap(v,u),\; Ymap(v,u)).
\end{aligned}
$$

Cała korekcja w runtime sprowadza się więc do jednego remapowania (resamplingu). To jest ważne, bo na pokładzie chcę mieć rozwiązanie proste i szybkie: mapa jest stała, a koszt przeliczenia pojedynczej klatki jest przewidywalny.
Kolaż poniżej przedstawia obraz szachownicy widziany z kamery, na którym oznaczono użyte punkty „obraz $\leftrightarrow$ droga”, fragment tablicy w wersji numerycznej oraz „wyprostowany” obraz po korekcji. Obraz po korekcji jest w skali: 1 piksel odpowiada 1 mm, a jego rozdzielczość to $840 \times 420$ pikseli.

<img src="{{ 'assets/images/Widok_z_lotu/F2.png' | relative_url }}" alt="F2" style="width:75%; max-width:100%; height:auto;" />

Warto zwrócić uwagę na dwa czarne trójkąty w lewym i prawym dolnym rogu — to obszary „poza mapą”, czyli miejsca, dla których nie da się wiarygodnie wskazać piksela źródłowego. Dodatkowo widać, że ostrość pól szachownicy nie jest w BEV idealnie równomierna. Obraz źródłowy ma równą gęstość pikseli, ale w widoku rybiego oka skrajne pola szachownicy zajmują mniej pikseli niż te bliżej środka kadru. W BEV oznacza to, że w niektórych obszarach algorytm musi „dopowiadać” szczegóły interpolacją — i tam obraz może być mniej ostry.


### 4 Przykład korekcji obrazu z przejazdu na macie

Kolaż poniżej przedstawia widoki w zastosowanym układzie drogi: widok z góry całej trasy, widok z kamery w wybranym punkcie, obraz po korekcji oraz miejsce na torze, z którego wykonano zdjęcie. Czerwona ramka określa obszar drogi widoczny na zdjęciu z kamery (i odpowiadający mu obszar po korekcji).
Na obrazie toru „z lotu ptaka” widać, że po korekcji przebieg krawędzi pasa jest znacznie bardziej regularny i stabilny w kolejnych klatkach. To ma bezpośrednie konsekwencje: taki obraz jest wygodny do budowy mapy przejazdu, a także do prostych algorytmów prowadzenia po linii. Dodatkowo BEV ułatwia uczenie sieci CNN do prowadzenia pojazdu, ponieważ sieć dostaje obraz w geometrii bliższej planowi podłogi, a nie w perspektywie rybiego oka

<img src="{{ 'assets/images/Widok_z_lotu/F3.png' | relative_url }}" alt="F3" style="width:75%; max-width:100%; height:auto;" />

### 5 Ograniczenia, o których trzeba pamiętać

To podejście działa dobrze dla płaszczyzny podłogi, ale ma oczywiste ograniczenia. Po pierwsze, mapowanie jest poprawne tylko dla konkretnego ustawienia kamery: wysokości, kąta nachylenia i obszaru widzenia. Po drugie, obiekty pionowe są w BEV przedstawione błędnie — i to nie jest wada implementacji, tylko konsekwencja założenia, że interesuje mnie płaszczyzna drogi

### 5 Co dalej 

W kolejnym kroku wykorzystam obraz w układzie drogi (w szczególności wariant krawędziowy) do złożenia wielu klatek w jedną monokularną mapę przejazdu. W tym miejscu bardzo przydaje się fakt, że odometria w moim pojeździe daje dobrą prognozę przesunięć i obrotów, a korekta wizyjna może pełnić rolę dopasowania i kompensacji poślizgów.
