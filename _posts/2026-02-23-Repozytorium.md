---
layout: post
title: "MODEL JĘZYKOWY NA POKŁADZIE SAMOCHODU"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Od rozmowy po polsku do sterowania pojazdem<!--more-->


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

### Czy samochód może mnie zrozumieć?
W poprzednich wpisach opisywałem kolejne elementy autonomii mojego samochodu: sterowanie napędem, odometrię, przetwarzanie obrazu z kamery i lokalizację z wykorzystaniem SLAM-u. Wszystkie te mechanizmy mają określone zadania i działają według zaimplementowanych algorytmów. Pozostaje jednak inne pytanie: czy samochód może zrozumieć polecenie wypowiedziane w języku naturalnym i samodzielnie wybrać odpowiednią funkcję do jego wykonania?

Postanowiłem sprawdzić to, uruchamiając lokalny model językowy (LLM) na komputerze pokładowym NVIDIA Jetson Orin Nano. Zależało mi na pracy bez połączenia z Internetem i możliwości komunikacji w języku polskim. Sam model nie miał bezpośrednio sterować silnikami ani serwomechanizmami. Jego zadaniem było prowadzenie rozmowy, interpretowanie poleceń i wskazywanie funkcji, które następnie wykonuje oddzielna warstwa programu.

Wybór modelu nie był oczywisty. Jetson ma ograniczone zasoby pamięci i mocy obliczeniowej, a niewielkie modele językowe różnią się szybkością działania, rozumieniem polskiego oraz możliwościami analizy obrazów i wywoływania funkcji. Pierwsze eksperymenty z ich uruchamianiem i porównaniem przeprowadził Michał, który opisał wyniki we wpisie [Tiny LLMs on the Edge – Jetson Nano](https://m1chol.github.io/m1/2025-09-23/jetson-llms). Na podstawie tych prac do dalszych prób wybraliśmy model Gemma 3 4B.

W tym artykule chcę przyjrzeć się trzem zagadnieniom. Najpierw pokażę kryteria wyboru lokalnego modelu i ograniczenia sprzętowe. Następnie sprawdzę jego możliwości: prowadzenie rozmowy, analizę obrazu oraz wybieranie dostępnych funkcji. Wreszcie przedstawię wyniki eksperymentów, od oceny konwersacji po sterowanie głosowe samochodem i pierwsze próby podejmowania decyzji na podstawie obrazu z kamery.

Najbardziej interesuje mnie granica między poprawnym zrozumieniem polecenia a jego wykonaniem. Model może wygenerować przekonującą odpowiedź, ale nie oznacza to jeszcze, że właściwie zinterpretuje zamiar operatora. A kiedy jego decyzja może spowodować fizyczny ruch pojazdu, ta różnica nabiera zupełnie praktycznego znaczenia.

### 2) Jak wybrać model językowy do samochodu?

Wybór modelu językowego można rozpocząć od porównania jego ogólnych możliwości. Dobrym punktem odniesienia jest [Artificial Analysis](https://artificialanalysis.ai/models/open-source/small), gdzie zestawiono wyniki testów niewielkich modeli otwartych.

W moim przypadku sama jakość modelu nie była jednak wystarczającym kryterium. Model miał działać lokalnie na NVIDIA Jetson Orin Nano, bez dostępu do Internetu, a jednocześnie umożliwiać swobodną komunikację po polsku.

Interesowały mnie również funkcje wykraczające poza zwykłą rozmowę. Model powinien umieć interpretować polecenia, generować odpowiedzi w określonym formacie, a docelowo także analizować obraz z kamery. Szczególnie ważna była możliwość wywoływania funkcji innych programów, ponieważ to ona pozwala połączyć model językowy z fizycznym działaniem samochodu.

Trzeba było przy tym uwzględnić ograniczenia sprzętowe. Większy model może uzyskiwać lepsze wyniki w testach, ale zajmować zbyt dużo pamięci lub odpowiadać zbyt wolno. Samochód potrzebuje przecież zasobów również do obsługi czujników i sterowania.

Porównanie wybranych modeli na Jetsonie przeprowadził Michał. Wyniki pomiarów szybkości generowania odpowiedzi, wykorzystania pamięci i oceny języka polskiego opisał we wpisie [Tiny LLMs on the Edge – Jetson Nano](https://m1chol.github.io/m1/2025-09-23/jetson-llms). Nie będę powtarzał tych testów.

Do dalszych eksperymentów wybraliśmy **Gemma 3 4B**, uruchamianą lokalnie przez Ollamę. Nie był to model o najwyższym ogólnym wyniku benchmarkowym ani najszybszy spośród testowanych. Okazał się jednak rozsądnym kompromisem między wymaganiami sprzętowymi, jakością rozmowy po polsku i dostępnymi możliwościami.

Pozostawało sprawdzić, jak ten wybór sprawdzi się w praktyce. Bo nawet model, który dobrze wypada w testach, może popełniać zaskakujące błędy podczas zwykłej rozmowy.

### 3) Rozumie, odpowiada, a czasem zmyśla

Pierwsze eksperymenty nie dotyczyły jeszcze sterowania samochodem. Chciałem po prostu sprawdzić, czy niewielki model językowy uruchomiony lokalnie na Jetsonie potrafi prowadzić rozmowę po polsku.

W większości prostych sytuacji odpowiedzi były poprawne. Model rozumiał pytania, potrafił kontynuować rozmowę i generować krótkie teksty. Pojawiły się jednak również przypadki, w których błędnie interpretował intencję użytkownika.

Dobrym przykładem jest następująca rozmowa:

**Użytkownik:** Cześć jek leci

**ATOM:** Dzień dobry! Sprawdzam aktualne informacje o lotach. Proszę, podaj numer lotu lub miejsce wylotu.

**Użytkownik:** LA1285

**ATOM:** Lot LA1285 z Los Angeles do Warszawy jest na czas. Oczekiwany czas przylotu: 06:30.

Wystarczyła literówka w potocznym pytaniu „Cześć, jak leci?”, aby model zinterpretował wypowiedź jako pytanie o połączenia lotnicze. Samo nieporozumienie nie byłoby szczególnie istotne, gdyby model poprosił o wyjaśnienie.

Tymczasem po otrzymaniu ciągu „LA1285” wygenerował szczegółową informację o trasie lotu, jego statusie i godzinie przylotu, mimo że nie miał dostępu do aktualnego systemu informacji lotniczej.

To przykład halucynacji: model wygenerował przekonująco brzmiącą odpowiedź bez danych, które mogłyby ją uzasadniać.

Podczas zwykłej rozmowy taka pomyłka bywa zabawna. W samochodzie sytuacja jest poważniejsza. Jeśli model błędnie zinterpretuje polecenie operatora, nie powinien mieć możliwości bezpośredniego uruchomienia dowolnego działania fizycznego.

### 4) Od słów do działania

Dotychczas ATOM był rozmówcą. Otrzymywał pytanie i generował odpowiedź, czasem poprawną, czasem zupełnie oderwaną od rzeczywistości. Postanowiliśmy sprawdzić, czy ten sam model może również interpretować polecenia dotyczące fizycznego działania samochodu.

Samo rozpoznanie słów nie wystarcza. Polecenie „skręć o czterdzieści pięć stopni” trzeba zamienić na informację zrozumiałą dla programu sterującego. Model językowy nie zna przy tym bezpośrednio sposobu komunikacji z serwomechanizmem ani nie powinien samodzielnie wysyłać sygnałów do urządzeń.

Dlatego zastosowaliśmy architekturę, w której LLM odpowiada za interpretację polecenia i wybór funkcji, natomiast jego wykonaniem zajmuje się oddzielny program.

Model otrzymuje instrukcję systemową, historię rozmowy i opis dostępnych funkcji. Może odpowiedzieć zwykłym tekstem albo zaproponować wywołanie funkcji wraz z argumentami, zapisane w formacie JSON.

Przykładowo polecenie:

**Użytkownik:** minus czterdzieści pięć stopni

zostało zamienione na:

```json
{"name": "set_angle", "parameters": {"angle": -45}}
```

Taka odpowiedź nie jest jeszcze komendą wysłaną do serwomechanizmu. Program najpierw sprawdza, czy funkcja znajduje się na liście dozwolonych działań, czy argumenty mają poprawną postać i mieszczą się w dopuszczalnych granicach. Dopiero po pozytywnej weryfikacji polecenie trafia do warstwy wykonawczej, która komunikuje się ze sterownikiem ESP32.

W eksperymencie udostępniliśmy modelowi ograniczony zestaw funkcji związanych ze sterowaniem i odczytem stanu urządzenia. Dzięki temu użytkownik mógł wydawać polecenia w języku naturalnym, nie znając nazw funkcji ani formatu komunikacji ze sterownikiem.

Ważne jest rozdzielenie odpowiedzialności. Model proponuje działanie, ale nie decyduje samodzielnie o tym, co wolno wykonać. Nieprawidłowa nazwa funkcji, błędny format odpowiedzi lub argument spoza dopuszczalnego zakresu powodują odrzucenie polecenia.

Tak powstał prosty agent z narzędziami. Ten sam model może prowadzić rozmowę i wybierać dostępne funkcje, ale dostęp do fizycznych elementów samochodu jest ograniczony przez niezależną warstwę programu.

<img src="{{ 'assets/images/LLM_1/schemat_1.png' | relative_url }}" alt="schemat_1" style="width:100%; max-width:75%; height:auto;" />

*Rys. 1. Interpretacja polecenia przez lokalny model językowy i jego wykonanie przez oddzielną, deterministyczną warstwę programu.*

## 5) Od polecenia głosowego do ruchu samochodu

Pierwsze próby wywoływania funkcji przeprowadziliśmy na pojedynczym serwomechanizmie. Model językowy otrzymywał polecenia wypowiadane po polsku i zamieniał je na wywołania funkcji `set_angle`. Pozwoliło to sprawdzić interpretację wartości liczbowych, wybór funkcji i współpracę z programem wykonawczym bez uruchamiania napędu samochodu.

Kolejnym krokiem było podłączenie agenta do sterowania pojazdem. Tym razem odpowiedź modelu mogła prowadzić do rzeczywistego ruchu samochodu, a nie tylko zmiany położenia pojedynczego serwomechanizmu.

Eksperymenty przeprowadziliśmy na zewnątrz, wykorzystując mikrofon, lokalny moduł rozpoznawania mowy, model Gemma 3 4B oraz program wykonawczy komunikujący się ze sterownikami pojazdu.

Najważniejsze stało się już nie samo wygenerowanie poprawnego polecenia, lecz zachowanie całego systemu: czy wypowiedź operatora została właściwie rozpoznana, czy model wybrał odpowiednią funkcję i czy samochód wykonał zamierzony manewr.

<img src="{{ 'assets/images/LLM_1/scena.png' | relative_url }}" alt="scena" style="width:100%; max-width:100%; height:auto;" />

*Rys. 2. Scena uliczna z robotem i otoczeniem.*

<img src="{{ 'assets/images/LLM_1/jazda 1.png' | relative_url }}" alt="jazda 1" style="width:100%; max-width:100%; height:auto;" />

*Rys. 3. Przebieg konwersacji i sterowania.*

