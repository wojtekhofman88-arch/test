# Asystent analizy wywiadów psychoterapeutycznych — Plan i Architektura

> Dokument roboczy. Zawiera wiążące ustalenia projektowe. Aktualizowany w miarę postępów.

## 1. Cel i zakres

Narzędzie **edukacyjne** wspierające (przyszłych) terapeutów — "kooterapeuta LLM".
Buduje **zespół hipotez** na podstawie transkrypcji sesji/wywiadu, by szybko poszerzać
wiedzę użytkownika.

**Wiążące zasady:**
- To NIE jest narzędzie do diagnozy klinicznej ani prawnej. Wynik to hipotezy/sygnały, nie rozstrzygnięcia.
- Każda odpowiedź zawiera zastrzeżenie o omylności.
- Człowiek (terapeuta) pozostaje w pętli i ponosi odpowiedzialność.
- Administrator danych: użytkownik (właściciel projektu). Wymagana zgoda pacjenta na nagrywanie i przetwarzanie AI.

## 2. Rama prawna (RODO / prawo polskie)

- Transkrypcja psychoterapii = **dane o zdrowiu psychicznym** = szczególna kategoria danych (art. 9 RODO).
- Usunięcie imion/miejsc to **pseudonimizacja** (art. 4 pkt 5), NIE anonimizacja — dane nadal podlegają RODO.
- Test identyfikowalności jest **oparty na ryzyku** ("means reasonably likely"), nie na 100% niemożliwości.
- Wniosek: dane przekazywane do API traktujemy jak pseudonimizowane dane szczególnej kategorii.
  Wymaga to: zgody, umowy powierzenia (DPA) z dostawcą API, opcji zero-retention (brak treningu),
  preferencji serwerów UE, minimalizacji danych.

## 3. Sprzęt i narzędzia

- **Mac mini M4, 16 GB** — realnie modele ~7-8B skwantyzowane (Bielik/Qwen) przez Ollama; jeden ciężki proces naraz.
- **Nagrywanie:** iPhone (Dyktafon), synchronizacja iCloud WYŁĄCZONA; transfer kablem/AirDrop; kasowanie z telefonu.
- **Transkrypcja:** MacWhisper (Pro — z rozróżnieniem mówców), model `large-v3-turbo`. W 100% lokalnie.
- **Orkiestracja:** OpenClaw (lokalny, autonomiczny runtime).
- **Modele specjalistyczne / wsparcie planowania:** API (na danych zanonimizowanych).
- **Wyjście:** Markdown jako minimum, docelowo czytelny PDF (generowany lokalnie).

## 4. Architektura: liniowy potok z agentowym rdzeniem rozumowania

```
======= LOKALNIE (dane wrażliwe) =======
[1] iPhone (Dyktafon, iCloud OFF)            -> audio
[2] MacWhisper (large-v3-turbo, diarization) -> transkrypcja.txt (Terapeuta/Pacjent)
[3] PSEUDONIMIZATOR (lokalny LLM + slownik + regex + human review)
        -> anon.txt  +  [sejf] klucz.json (przechowywany osobno, lokalnie)

------- GRANICA PRYWATNOSCI: dalej tylko dane zanonimizowane -------

======= API (dane zanonimizowane, DPA + zero-retention) =======
[4] DYRYGENT (lokalny OpenClaw, autonomiczny)
     - czyta "zywa konceptualizacje" (wspolny stan sprawy)
     - petla: plan -> wybor skilla -> wynik {ustalenia, pewnosc, dowody, kolejne_skille}
       -> aktualizacja stanu -> ... az do warunku stopu
     - eskalacja planowania do API wg regul (np. niska pewnosc / konkurujace diagnozy)
     - skille: diagnoza ogolna -> pogłębiające, mysli automatyczne,
               glowny problem, konceptualizacja CBT, plan terapii
     - [ZAWSZE] skill: detekcja ryzyka (samobojcze/zagrozenie) -> na wierzch raportu
     - [NA KONCU] superwizorzy (INNY model) -> recenzja -> poprawki

======= LOKALNIE (powrot do prawdziwych danych) =======
[5] raport + [sejf] klucz -> re-identyfikacja -> raport_finalny.pdf (zostaje lokalnie)
```

## 5. Wiążące usprawnienia architektury

1. **Granica prywatności = pseudonimizacja.** Tylko anonimizator + sejf z kluczem muszą być lokalne.
   Wszystko po tej granicy działa na danych zanonimizowanych.
2. **Dyrygent: lokalny OpenClaw**, autonomiczny. Sam decyduje o eskalacji do API jako jednym z narzędzi.
   Wiarygodność zapewniają kontrakty skilli + reguły eskalacji, nie sama "inteligencja" modelu 7B.
3. **Kontrakty skilli (ustrukturyzowane I/O, JSON):**
   `wejscie: {fragmenty, dotychczasowe_ustalenia}`
   `wyjscie: {ustalenia, pewnosc (0-1), cytaty_dowody, rekomendowane_kolejne_skille}`
4. **Wspólny stan sprawy** — "żywa konceptualizacja": rosnący, ustrukturyzowany dokument roboczy
   (hipotezy, dowody, pytania otwarte), na podstawie którego dyrygent planuje kolejne kroki.
5. **Pętla plan -> działanie -> ocena z warunkiem stopu** (budżet wywołań / próg pewności / brak nowych informacji).
6. **Zasada "no evidence, no claim"** — każde ustalenie cytuje konkretny fragment transkrypcji.
7. **Zawsze włączony moduł detekcji ryzyka**, niezależny od zapytania użytkownika.
8. **Recenzja przez inny model** niż generujący (unikanie echa).
9. **Pseudonimizacja trójwarstwowa:** (a) słownik + regex (deterministycznie), (b) LLM flaguje wątpliwe konteksty,
   (c) obowiązkowy human-review przed wyjściem czegokolwiek poza maszynę.
10. **Zestaw testowy (gold standard)** — kilka transkrypcji z wzorcową odpowiedzią do oceny jakości zmian.

## 6. Zadania użytkownika (przykładowe)

- "Wskaż negatywne myśli automatyczne związane z głównym problemem pacjenta."
- "Postaw hipotezy diagnostyczne (jako sygnały, nie diagnozę)."
- "Zbuduj konceptualizację CBT pacjenta."
- "Zaplanuj kolejne kroki terapii."

## 7. Plan etapowy (kolejność wdrażania)

- **FAZA A — w 100% lokalna, zero ryzyka prawnego, fundament:**
  - Etap 1: nagrywanie (iPhone) + transfer
  - Etap 2: transkrypcja (MacWhisper)
  - Etap 3: pseudonimizacja (słownik + regex + LLM + human review)
- **FAZA B — wejście API (tylko dane zanonimizowane):**
  - Etap 4 z JEDNYM zadaniem na start (np. myśli automatyczne), kontrakty skilli, stan sprawy
- **FAZA C — jakość i wygoda:**
  - reszta skilli, eskalacja, superwizorzy, re-identyfikacja, PDF, zestaw testowy

## 8. Otwarte kwestie

- Dobór dostawcy API + DPA + lokalizacja serwerów (UE).
- Format/struktura klucza pseudonimizacji (odwracalność po stronie lokalnej).
- Reguły eskalacji do API (progi pewności).
- Czy "żywa konceptualizacja" + RAG po wielu sesjach jednego pacjenta od razu, czy później.
