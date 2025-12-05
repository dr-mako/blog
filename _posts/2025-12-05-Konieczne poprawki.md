---
layout: post
title: "Niezbędne poprawki konstrukcyjne"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

### Kierunek zmian: pełna kinematyka Ackermanna<!--more-->


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


Analiza modeli i testy potwierdziły potrzebę modernizacji: aby uzyskać prawidłową kinematykę Ackermanna, należy zastąpić obecną konfigurację 4WS (para kół przód/tył o tym samym kącie) układem 4 kół skrętnych niezależnie. Modernizacja jest prosta — na początku projektu przewidziałem tę opcję.

Plan działania:


- Wykonać dwie nowe skrajne części płyty górnej nośnej z otworami pod serwa (wydruk 3D).

- Otrzymuje każde koło sterowne niezależnie prędkością (silniki) i kątem (serwo) - pasek rozrządu do przeniesienia skretu na sparowaną oś kół będzie niepotrzebny.

- Zasilanie pozostaje bez zmian — zapas mocy jest wystarczający.

- zmodyfikować program dla nowej funkcjonalności - implementacji dyferencjału kątowego kół (realizacja pełnej kinematyki Ackermanna).