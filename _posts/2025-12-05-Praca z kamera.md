---
layout: post
title: "Przejazd z kamerą"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### ArduCAM IMX219 Wide Angle Camera Module<!--more-->


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


### 1) Rejestruje obrazy z kamery

W systemie używam modułu „ArduCAM IMX219 Wide Angle Camera” z obiektywem typu rybie oko 170°. Wybrałem go, ponieważ jest wspierany przez Nvidię dla Jetsona Orin Nano, którego używam. Muszę zaznaczyć, że uruchomienie kamery było wyjątkowo trudne i czasochłonne — pierwszy raz spotkałem się z tyloma niuansami. Obecnie kamera pracuje w trybie zdjęciowym 1640×1232 przy 30 fps. Równolegle obrazy są streamowane na stronę WWW, co pozwala na podgląd „na żywo”.

### 2) Jak to teraz wygląda - czyli gdzie jestem z projektem

Na kolażu poniżej pokazuję aktualny stan:


- Kolumna lewa, góra: fotografia toru jazdy. Do testów używam maty z wyznaczonymi pasami.

- Kolumna lewa, dół: trajektoria i błędy szacowania pozycji wyznaczane wyłącznie z odometrii (model „Ackermann predict”). Widać bardzo dobre dopasowanie.

- Kolumna środkowa, góra: surowe logi z enkoderów i silników (podstawa do odometrii).

- Kolumna środkowa, dół: nowość — seria klatek z kamery na pojeździe, czyli „jak pojazd widzi trasę”.

- Kolumna prawa: dwa powiększone zrzuty z przejazdu.


<img src="{{ 'assets/images/kamera/kamera.png' | relative_url }}" alt="kamera" style="width:100%; max-width:100%; height:auto;" />

### 3 Z jakim problemem się spotkałem i jakie to niesie konsekwencję

Używam taśmy CSI do kamery która ma długość 15 cm. Próba z taśmą 30 cm zakończyła się brakiem obrazu i nagrzewaniem taśmy, więc zostałem przy 15 cm. To wymusza niskie osadzenie kamery. Podczas ostrych zakrętów tracę widoczność zewnętrznych pasów, co utrudnia segmentację i późniejsze decyzje sterujące. 

### 4 co dalej?

- Budowa mapy obrazów. Korekta perspektywy "fisheye" do widoku z lotu ptaka "bird's eye view". Do łączenia klatek rozważam RANSAC (detekcja i dopasowanie cech, odrzucanie outlierów), a następnie złożenie mozaiki.

- Fuzja map: spróbuję scalić mapę odometryczną i wizyjną metodą bayesowską (ważenie wiarygodności źródeł).

- Autonomia: planuję dotrenować sieć CNN „AlexNet” do autonomicznego prowadzenia po torze metodą transfer learning z uczeniem nadzorowanym. Etykiety to nastawy prędkości i kąta skrętu.

- Agent LLM: demonstrator sterowania urządzeniami peryferyjnymi przez lokalny model LLM na Jetsonie (np. Gemma 3 lub równoważny) — wydawanie komend prędkości/skrętu oraz analiza zdjęć z kamery.

