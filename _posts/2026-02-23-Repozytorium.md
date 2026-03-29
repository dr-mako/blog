---
layout: post
title: "REPOZYTORIUM PROJEKTU"
author: "Maciej Kozłowski"
excerpt_separator: <!--more-->
---

Ten blog składa sie z następujących wpisów:
- Wprowadzenie "Model pojazdu do badań nad autonomią"
- Opis konstrukcji "Etapy budowy platformy modelu"
- Modelowanie "Aktualizacja położenia w oparciu o model i metoda wyznaczania trajektorii ruchu"
- Pierwsze przejazdy "Surowe wyniki i wyznaczanie trajektorii w postprocesingu"
- Jak rośnie błąd trajektorii? Ackermann‑predict w praktyce czyli Rachunek niepewności i propagacja błędu w modelu ruchu
- Testy dokładności "Pierwsza seria ćwiczeń. Przejazdy po torze modułowym i weryfikacja szacowania pozycji"
- Przejazd z kamerą czyli "co możemy odczytać ze zdjęć"
- Korekta obrazu "Metody tworzenia bird’s eye view (BEV)"
- "Wykrywanie pasów ruchu w BEV (bird’s eye view)"
- Map fitting, co to jest i dlaczego to “nie działa”? - "Kamera jako czujnik położenia"
- Kiedy trajektoria wreszcie może się poprawić? "SLAM offline"


Jeśli mój projekt Cię zaciekawił ... 

<!--more-->

możesz zajrzeć do repozytorium. Umieściłem tam cały kod używany w kolejnych etapach opisywanych na blogu: transformację obrazu z kamery (FEV → BEV), map fitting oraz SLAM offline (post‑processing na logach przejazdów), a także skrypty do synchronizacji logów, ekstrakcji obserwacji i diagnostyki wyników.

Repozytorium znajdziesz tutaj: https://github.com/dr-mako/samochod

W repo jest też krótki README.md, który prowadzi przez strukturę katalogów i opisuje, do czego służą poszczególne skrypty oraz jakie pliki są wejściem/wyjściem w pipeline. Jeśli chcesz odtworzyć eksperymenty krok po kroku, zacznij właśnie od tego pliku.