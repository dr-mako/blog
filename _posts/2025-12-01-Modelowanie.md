---
layout: post
title: "Aktualizacja położenia w oparciu o model i wyznaczanie trajektorii ruchu"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Opis modelu <!--more-->

Parametry geometryczne prototypu

- Rozstaw kół: \$$ D = 153\,\mathrm{mm} $$

- Rozstaw osi: \$$ L = 260\,\mathrm{mm} $$

- Zakres skrętu mechaniczny: \$$ 0^\circ\!-\!90^\circ$, ograniczenie programowe: $35^\circ $$

- Promień koła: \$$ R_w = 37.25\,\mathrm{mm} $$



### 1) Koncepcja
Zastosuję prosty model Ackermanna w ujęciu „rowerowym” (bicycle model) dla konfiguracji 2WS (przednia para kół sketna), a następnie rozszerzę go do mojego przypadku 4WS (przód + tył skrętne, przeciwfazowo).

W modelu 2WS skrętna jest wyłącznie oś przednia, a oś tylna jest toczna. W idealizacji Ackermanna dopuszczamy różne kąty skrętu kół wewnętrznego i zewnętrznego na osi przedniej tak, aby przedłużenia ich płaszczyzn przecinały się w jednym punkcie ICR (środek obrotu - Instantaneous Center of Rotation). Dzięki temu ruch jest bezpoślizgowy: wszystkie koła poruszają się po współśrodkowych okręgach ze wspólnym środkiem obrotu ICR.

Model „rowerowy” zastępuje pary kół kołem ekwiwalentnym: z przodu i z tyłu umieszczam po jednym „kole zastępczym” w środku osi, a skręt opisuję pojedynczym kątem $\delta$. Ponieważ w modelu 2WS łuk ruchu wyznacza oś tylna, stan pojazdu definiuję w środku osi tylnej: zmienne stanu $[x, y, \Theta]$ to globalne współrzędne i orientacja pojazdu. Promień do ICR oznaczam $R_{\mathrm{ICR}}$.

Ten model pozwoli pokazać, że sterowanie pojazdem mogę realizować w lokalnym układzie odniesienia za pomocą pary zmiennych $[v, \delta]$ (prędkość i kąt skrętu), a jego efektem jest zmiana położenia i kursu w układzie globalnym, opisana przez zmienne $[x, y, \Theta]$. Na tej bazie w prosty sposób rozszerzę opis do przypadku 4WS.

#### 2) Model 2WS. Minimum wzorów do opisania zależności kinematycznych
Stosuje model 2WS "rowerowy". Przyjmuję następujące założenia:
- Koła tylne są sztywne (oś tylna toczna, bez skrętu).
- Koła przednie są skrętne; dopuszczamy różne kąty kół wewnętrznego i zewnętrznego, w ten sposób że proste prostopadłe do powierzchni bocznych przecinają się w jednym puncie - środku obrotu pojazdu ICR.

<img src="{{ 'assets/images/modelowanie/model2WS.png' | relative_url }}" alt="model2WS" style="width:75%; max-width:100%; height:auto;" />

Otrzymuje:
Związek krzywizny z kątem skrętu:

$$ R_{\mathrm{ICR}} = \dfrac{L}{\tan \delta} $$

Długość przebytej drogi w kroku czasu $\Delta T$ w ruchu po wycinku łuku kołowego wynoszą: 

$$\lambda = v\,\Delta T = R_{\mathrm{ICR}}\,\Delta \theta$$

Oznaczenia:

\(L\) — rozstaw osi [mm]
\(D\) — rozstaw kół (szerokość toru) [mm]
\(R_w\) — promień koła [mm]
\(\delta\) — kąt skrętu w modelu „rowerowym” (ekwiwalent dla osi przedniej) [rad]
\(R_{\mathrm{ICR}}\) — promień do chwilowego środka obrotu (ICR) mierzony od środka osi tylnej [mm]
\(v\) — prędkość liniowa środka osi tylnej [mm/s]
\(\lambda\) — długość przebytego łuku w kroku \(\Delta T\) [mm]
\(\Delta \theta\) — przyrost orientacji w kroku \(\Delta T\) [rad]
\([x, y, \Theta]\) — zmienne stanu: współrzędne i orientacja w układzie globalnym

Równania ruchu (postać ciągła dla punktu środka osi tylnej, układ globalny) przyjmują postać:

\(\dfrac{dx}{dt} = v \cos \Theta\)

\(\dfrac{dy}{dt} = v \sin \Theta\)

\(\dfrac{d\Theta}{dt} = \dfrac{v}{L}\,\tan \delta\)

gdzie oznaczono:

\(x, y\) — położenie środka osi tylnej w układzie globalnym
\(\Theta\) — orientacja pojazdu w układzie globalnym
\(v\) — prędkość liniowa wzdłuż osi pojazdu (lokalnie)
\(\delta\) — kąt skrętu w modelu „rowerowym” (ekwiwalent dla osi przedniej)
\(L\) — rozstaw osi

#### 3) Model geometrii samochodu 4WS — skręt przeciwfazowy
Pojazd obraca się wokół punktu przecięcia promieni skrętu obu osi (Instantaneous Center of Rotation, ICR). Dla skrętu przeciwfazowego (katy skrętu pary kół przód i tył o tej samej wartości, lecz przeciwnie skierowanej) ICR leży na osi symetrii pojazdu, a promień toru środka pojazdu wynosi:

\(R_{\mathrm{ICR}} = \dfrac{L}{\tan \delta + \tan(-\delta)} = \dfrac{L}{2\,\tan \delta}\)

Długość przebytej drogi w kroku czasu \(\Delta T\) w ruchu po wycinku łuku kołowego wynoszą: 

\(\lambda = v\,\Delta T = R_{\mathrm{ICR}}\,\Delta \theta\)

<img src="{{ 'assets/images/modelowanie/model4WS.png' | relative_url }}" alt="model4WS" style="width:50%; max-width:100%; height:auto;" />

Równania ruchu (postać ciągła dla środka pojazdu, układ globalny) przyjmuja postać:


\(\dfrac{dx}{dt} = v \cos \Theta\)

\(\dfrac{dy}{dt} = v \sin \Theta\)

\(\dfrac{d\Theta}{dt} = \dfrac{2v}{L}\,\tan \delta\)

Przyjmując ruch bez poślizgu (uproszczenie konieczne dla zachowania "czystej" kinematyki), promienie toru kół (względem tego samego ICR) określam następująco:

wewnętrzny: \(R_{\mathrm{in}}(\delta) = R_{\mathrm{ICR}}(\delta) - \dfrac{D}{2}\)

zewnętrzny: \(R_{\mathrm{out}}(\delta) = R_{\mathrm{ICR}}(\delta) + \dfrac{D}{2}\)

Dla sterowania „prędkością centralną” (\(n_c \propto v\)) otrzymuję dyferencjał prędkościowy (prędkości kół dobrane do promieni toru, aby unikać poślizgu):

\(n_{\mathrm{in}} = n_c \dfrac{R_{\mathrm{in}}}{R_c},\quad n_{\mathrm{out}} = n_c \dfrac{R_{\mathrm{out}}}{R_c}\)


#### 4) Model 4WS pary kół identycznie skrętne — różnica względem Ackermanna
W praktycznej realizacji mojego pojazdu przyjmuję równoległe ustawienie kół po lewej i prawej stronie tej samej osi (brak geometrii Ackermanna na osi). To oznacza:

- kąty kół lewe/prawe na danej osi są identyczne,
- proste prostopadłe do płaszczyzn kół na tej osi są równoległe i nie przecinają się w jednym punkcie,
- nie istnieje jeden wspólny ICR dla wszystkich kół.

Konsekwencje:

- pojawiają się poślizgi wiertne (lateral scrub) i momenty obrotowe,
- rzeczywiste położenie „efektywnego” ICR oraz rozkład prędkości/poślizgów wynikają z sił kontaktowych opona–podłoże,
- aby je ująć, potrzebny był by model dynamiki opon (np. Pacejka/Magic Formula) lub przynajmniej quasi‑statyczny model sił bocznych.

#### 5) Obserwacja zmiennych stanu — problem do rozwiązania
Jak pokazałem wcześniej, ruch pojazdu opisany zmiennymi stanu \([x, y, \Theta]\) traktuję jako konsekwencję sterowania parami \([v, \delta]\). Teraz zaglądam „pod maskę” — chcę zobaczyć, co dzieje się, gdy pojawią się błędy i zakłócenia oraz jak wpływają one na proces sterowania (którego celem jest osiągnięcie pożądanego zachowania obiektu mimo zakłóceń).

<img src="{{ 'assets/images/modelowanie/Schemat_sterowania.png' | relative_url }}" alt="Schemat_sterowania" style="width:120%; max-width:100%; height:auto;" />

Rysunek powyżej pokazuje, że szumy i zakłócenia (według miejsca powstawania w łańcuchu sterowania) można rozróżnić następująco:

- sterowania — dodają się do sygnałów sterujących (gaz/hamulec/skręt),
- procesowe — wpływają na sam obiekt (np. efektywny moment napędowy, kąt skrętu),
- pomiarowe — zanieczyszczają wynik pomiaru z czujników.

W moim układzie mogę bezpośrednio nastawiać i mierzyć prędkości obrotowe kół oraz kąty skrętu serw. Natomiast zmienne stanu opisujące ruch (położenie i kurs) nie są mierzalne wprost. Jeśli na nich mi zależy — a to jest właśnie ten przypadek — muszę je wyznaczać pośrednio na podstawie równań modelu. Problemem są tu zakłócenia i szumy. Tę funkcję przejmie obserwator zmiennych stanu.

<img src="{{ 'assets/images/modelowanie/obserwator.png' | relative_url }}" alt="obserwator" style="width:75%; max-width:100%; height:auto;" />

Mój model, w którym umieszczam obserwator, ma dwie warstwy zmiennych: widzialną i ukrytą.

Warstwa widzialna:
- sterowania zadane \(u\) (gaz/hamulec/skręt), obarczone szumem sterowania,
- pomiary \(y\) (z czujników), obarczone szumem pomiarowym.

Warstwa ukryta:
- rzeczywisty stan \(x\) (położenie, orientacja/kurs, prędkości, ewentualne poślizgi), podlegający niepewnościom: realizacyjnym, procesowym i pomiarowym.

<img src="{{ 'assets/images/modelowanie/warstwy_modelu.png' | relative_url }}" alt="warstwy_modelu" style="width:75%; max-width:100%; height:auto;" />

Cel modelowania formułuję następująco: wyznaczyć najlepsze oszacowanie \(\hat{x}\) na podstawie \(u\) i \(y\), mimo błędów modelu i szumów. Jest to problem probabilistyczny; zastosuję prawo propagacji niepewności i filtrację.

Ujęcie równań modelu:
- dynamika: \(x_{k+1} = f(x_k, u_k, d_k) + w_k\)  — szum procesu \(w_k\),
- pomiar: \(y_k = h(x_k) + v_k\)  — szum pomiaru \(v_k\).

Praktyka: obserwator/filtr (np. filtr Kalmana) łączy model z pomiarem, równoważąc niepewność wnoszoną przez \(w\) i \(v\), tak aby na bieżąco dostarczać estymatę stanu \(\hat{x}_k\) użyteczną do sterowania i diagnostyki.


#### 6) Prosta odometria modelu 4WS
Zanim zacznę modelowanie, ustalam sposób sterowania i efekt ruchu oraz wybór modelu. Pojazdem steruję, nadając prędkości silnikom i ustawiając kąt skrętu kół. Efektem jest ruch postępowy albo obrót po łuku wokół chwilowego środka obrotu ICR. Do opisu wystarczy najprostsza kinematyka — model Ackermanna w wariancie 4WS (przeciwfazowo), bez wchodzenia w dynamikę i poślizgi.

Stan pojazdu opisuję wektorem x = [x, y, Θ]^T, gdzie x i y to współrzędne w globalnym układzie odniesienia, a Θ to orientacja (kąt zwrotu) nadwozia względem osi OX. Sterowanie zbieram w wektorze u = [v_icr, δ]^T. W praktyce wygodniej jest sterować prędkością liniową pojazdu v (w środku pojazdu) i kątem skrętu δ, ale w odometrii 4WS można myśleć równoważnie o prędkości „centralnej” związanej z ruchem po okręgu wokół ICR.

Równania aktualizacji stanu otrzymuję przez dyskretyzację równań ruchu. Stosuję prostą dyskretyzację explicite (Euler w przód) z kątem liczonym z poprzedniego kroku, bo w typowym sterowaniu wartości utrzymane są stałe przez cały krok czasu ΔT. Wtedy:

\(x_k = x_{k-1} + V_k\,\Delta T \,\cos \Theta_{k-1}\)
\(y_k = y_{k-1} + V_k\,\Delta T \,\sin \Theta_{k-1}\)
\(\Theta_k = \Theta_{k-1} + \dfrac{2 V_k\,\Delta T}{L}\,\tan \delta_k\)

gdzie: \(\delta_k\) to zastępczy kąt skrętu w modelu 4WS przeciwfazowego, \(\Theta_k\) to orientacja pojazdu w układzie globalnym, \(V_k\) prędkość liniowa (w środku pojazdu), \(L\) rozstaw osi, a \(\Delta T\) krok czasu. Ten schemat stanowi bazę do odometrii: integruję przebyte odcinki i przyrosty orientacji, korzystając z próbkowanych sygnałów sterujących i/lub z estymowanych prędkości. W praktycznej implementacji ograniczam kąt skrętu do dopuszczalnego zakresu, pilnuję wspólnego zegara dla wszystkich sygnałów oraz — gdy to potrzebne — stosuję korekty (np. filtrację) w celu redukcji dryftu wynikającego z szumów i błędów modelu.

#### 7) Co dalej
W następnej sekcji opiszę wyniki pierwszych jazd i poddam je krytycznej analizie