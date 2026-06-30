# RAPORT PROJEKTU — Asystent analizy wywiadów psychoterapeutycznych

> Pełne podsumowanie ustaleń z konwersacji. Pojedynczy punkt odniesienia.
> (Wątek incydentu bezpieczeństwa pominięty — sprawa zamknięta.)

---

## 1. Cel i zakres

Narzędzie **edukacyjne** wspierające (przyszłych) terapeutów — "kooterapeuta LLM".
Na podstawie transkrypcji sesji/wywiadu buduje **zespół hipotez**, by szybko poszerzać wiedzę użytkownika.

**Wiążące zasady:**
- To NIE jest narzędzie do diagnozy klinicznej ani prawnej — wynik to **hipotezy/sygnały**, nie rozstrzygnięcia.
- Każda odpowiedź zawiera **zastrzeżenie o omylności**.
- **Człowiek (terapeuta) zostaje w pętli** i ponosi odpowiedzialność.
- Administrator danych: użytkownik (właściciel projektu). Wymagana **zgoda pacjenta** na nagrywanie i przetwarzanie AI.
- Zastosowanie: niekomercyjne, edukacyjne.

---

## 2. Rama prawna (RODO / prawo polskie)

- Transkrypcja psychoterapii = **dane o zdrowiu psychicznym** = szczególna kategoria danych (art. 9 RODO).
- Usunięcie imion/miejsc to **pseudonimizacja** (art. 4 pkt 5), NIE anonimizacja — dane nadal podlegają RODO.
- Identyfikowalność jest **pośrednia** (art. 4 pkt 1): kombinacja czynników (objawy, przeszłość, zawód) identyfikuje osobę nawet bez nazwiska.
- Test identyfikowalności jest **oparty na ryzyku** ("means reasonably likely", motyw 26) — nie na 100% niemożliwości. 100% anonimowości się nie da, ale liczy się ryzyko realistyczne.
- **Wniosek:** dane przekazywane do API traktujemy jak pseudonimizowane dane szczególnej kategorii. Wymaga to: zgody, umowy powierzenia (DPA), opcji zero-retention (brak treningu na danych), preferencji serwerów UE, minimalizacji danych.

---

## 3. Architektura — liniowy potok z agentowym rdzeniem rozumowania

Przepływ ZEWNĘTRZNY jest liniowy; rdzeń ROZUMOWANIA (krok 4) jest dynamiczny/agentowy.

```
======= LOKALNIE (dane wrażliwe) =======
[1] iPhone (Dyktafon, iCloud OFF)            -> audio
[2] Transkrypcja (WhisperX/whispermlx)       -> transkrypcja.txt (Terapeuta/Pacjent)
[3] PSEUDONIMIZATOR (lokalny LLM + slownik + regex + human review)
        -> anon.txt  +  [sejf] klucz.json (osobno, lokalnie)

------- GRANICA PRYWATNOSCI: dalej tylko dane zanonimizowane -------

======= API (dane zanonimizowane, DPA + zero-retention) =======
[4] DYRYGENT (lokalny OpenClaw, autonomiczny)
     - czyta "zywa konceptualizacje" (wspolny stan sprawy)
     - petla: plan -> wybor skilla -> wynik {ustalenia, pewnosc, dowody, kolejne_skille}
       -> aktualizacja stanu -> ... az do warunku stopu
     - eskalacja do API wg regul (niska pewnosc / konkurujace diagnozy / limit tokenow)
     - skille: diagnoza ogolna -> poglebiajace, mysli automatyczne,
               glowny problem, konceptualizacja CBT, plan terapii
     - [ZAWSZE] skill: detekcja ryzyka (samobojcze/zagrozenie) -> na wierzch raportu
     - [NA KONCU] superwizorzy (INNY model) -> recenzja -> poprawki

======= LOKALNIE (powrot do prawdziwych danych) =======
[5] raport + [sejf] klucz -> re-identyfikacja -> raport_finalny.pdf (zostaje lokalnie)
```

---

## 4. Kroki przetwarzania (input -> output)

1. Transfer audio z iPhone na Maca (kabel/AirDrop), iCloud OFF.
2. Transkrypcja -> tekst z oznaczeniem mówców (Terapeuta/Pacjent).
3. Pseudonimizacja -> tekst zanonimizowany + klucz (sejf lokalny). **GRANICA PRYWATNOŚCI.**
4. Orkiestracja: dyrygent dobiera skille -> raport roboczy (hipotezy + dowody z cytatami; moduł ryzyka aktywny).
5. Recenzja: superwizorzy (inny model) -> poprawki.
6. Re-identyfikacja (klucz) + render -> raport_finalny.pdf (lokalnie).

---

## 5. Ogólne zasady (obowiązują wszędzie)

1. **Granica prywatności = krok 3.** Surowe audio i pełna transkrypcja nigdy nie opuszczają Maca.
2. **Człowiek w pętli** w dwóch miejscach: zatwierdzenie flagowanych fragmentów (krok 3), odbiór raportu (krok 5).
3. **"No evidence, no claim"** — każde ustalenie cytuje fragment transkrypcji.
4. **Hipotezy, nie diagnozy** — zastrzeżenie o omylności w każdym wyjściu.
5. **Moduł ryzyka zawsze aktywny** — niezależnie od zapytania.
6. **Każdy moduł ma kontrakt (wejście/wyjście)** — testowalny w izolacji.
7. **Każda zmiana mierzona** — porównanie do wzorca, by nie pogorszyć jakości po cichu.

---

## 6. Architektura modułów (testowalne osobno)

| Moduł | Wejście | Wyjście | Rola |
|---|---|---|---|
| M1 — Pozyskanie audio | nagranie iPhone | plik audio na Macu | transfer, kontrola iCloud |
| M2 — Transkrypcja | plik audio | tekst z mówcami | WhisperX/whispermlx |
| M3 — Pseudonimizacja | tekst | tekst zanon + klucz | słownik+regex -> LLM flaguje -> human review |
| M4 — Orkiestracja | tekst zanon + polecenie | raport roboczy | dyrygent + skille |
| M5 — Recenzja | raport roboczy | raport poprawiony | superwizorzy (inny model) |
| M6 — Re-identyfikacja + PDF | raport + klucz | raport_finalny.pdf | podstawienie danych, render |

Podmoduły M4: M4a pojedyncze skille, M4b dyrygent/routing, M4c stan sprawy, M4d moduł ryzyka.

---

## 7. Wiążące usprawnienia architektury

1. **Granica prywatności = pseudonimizacja.** Tylko anonimizator + sejf z kluczem muszą być lokalne.
2. **Dyrygent: lokalny OpenClaw**, autonomiczny; sam decyduje o eskalacji do API jako narzędziu.
   Wiarygodność daje mu struktura (kontrakty + reguły), nie sama "inteligencja" modelu 7B.
3. **Kontrakty skilli (JSON):** `wejscie {fragmenty, ustalenia}` -> `wyjscie {ustalenia, pewnosc, cytaty_dowody, rekomendowane_kolejne_skille}`.
4. **Wspólny stan sprawy** — "żywa konceptualizacja": rosnący dokument roboczy (hipotezy, dowody, pytania).
5. **Pętla plan -> działanie -> ocena z warunkiem stopu** (budżet / próg pewności / brak nowych informacji).
6. **"No evidence, no claim"** — cytat z transkrypcji do każdego ustalenia.
7. **Zawsze włączony moduł detekcji ryzyka**, niezależny od zapytania.
8. **Recenzja przez inny model** niż generujący (unikanie echa).
9. **Pseudonimizacja trójwarstwowa:** (a) słownik + regex deterministycznie, (b) LLM flaguje wątpliwe konteksty, (c) obowiązkowy human-review przed wyjściem poza maszynę.
10. **Zestaw testowy (gold standard)** — transkrypcje wzorcowe do oceny jakości zmian.
11. **Sterowanie limitem tokenów** — dyrygent dostosowuje listę skilli do limitu narzuconego przez użytkownika (ważne przy API/koszt).

---

## 8. Stack i sprzęt

- **Mac mini M4, 16 GB** — realnie modele ~7-8B skwantyzowane przez Ollama; jeden ciężki proces naraz.
- **Nagrywanie:** iPhone (Dyktafon), iCloud OFF; transfer kablem/AirDrop; kasowanie z telefonu.
- **Transkrypcja:** patrz decyzja w sekcji 9.
- **Orkiestracja:** OpenClaw (lokalny, autonomiczny runtime; interfejs np. czat).
- **Modele lokalne:** Bielik/Qwen 7B.
- **API:** mocny model do trudnego rozumowania klinicznego i wsparcia planowania (tylko dane zanon.).
- **Wyjście:** Markdown (minimum) -> docelowo czytelny PDF (lokalnie).

**Zapotrzebowanie RAM:**
| Scenariusz | RAM | Uwaga |
|---|---|---|
| Plan 1.0 (hybryda, sekwencyjnie) | 16 GB | wystarczy, ale na styk; jeden ciężki proces naraz |
| Komfort (lokalny dyrygent 14B) | 24-32 GB | lepsza jakość dyrygowania |
| Cel "wszystko lokalnie" (70B) | 48 GB+ | docelowo, bez API |

16 GB starcza dla 1.0, bo ciężkie rozumowanie idzie do API (zero lokalnego RAM).

---

## 9. Decyzje narzędziowe (transkrypcja)

**Decyzja:** Próbujemy **WhisperX** (rodzina), z preferencją wersji zoptymalizowanej pod Apple Silicon.
- **1. wybór na M4:** `whispermlx` (MLX = GPU Apple, szybsze) — darmowe, diarization, skryptowalne.
- **2. wybór (fallback):** klasyczny `whisperx` (CPU na Macu, wolniejsze).
- **Plan B:** **MacWhisper** (łatwy GUI, szybki na Apple Silicon; diarization w wersji Pro płatnej, trudny do automatyzacji) — jeśli rodzina WhisperX zawiedzie.

**Uzasadnienie:** WhisperX jest darmowy, ma rozdzielanie mówców i jest **skryptowalny** (pasuje do automatyzacji w OpenClaw). MacWhisper łatwiejszy, ale GUI = trudny do automatyzacji.

**Bezpieczna instalacja:** tylko z zaufanych źródeł — App Store / oficjalne PyPI / GitHub / ręcznie wpisany `macwhisper.com`. Nigdy z linków/podróbek.

**Model transkrypcji:** Whisper `large-v3` / `large-v3-turbo` (turbo lżejszy i szybszy, mieści się w 16 GB).

**Diarization (pyannote) wymaga:** darmowego konta Hugging Face + akceptacji licencji modeli `pyannote/speaker-diarization-3.1` i `pyannote/segmentation-3.0` + tokenu dostępu. Służy WYŁĄCZNIE do pobrania modelu — audio nigdzie nie jest wysyłane, diarization działa lokalnie.

---

## 10. Plan etapowy (kolejność wdrażania)

- **FAZA A — w 100% lokalna, zero ryzyka prawnego, fundament:**
  - Etap 1: nagrywanie (iPhone) + transfer
  - Etap 2: transkrypcja
  - Etap 3: pseudonimizacja (słownik + regex + LLM + human review)
- **FAZA B — wejście API (tylko dane zanonimizowane):**
  - Etap 4 z JEDNYM zadaniem na start (np. myśli automatyczne), kontrakty skilli, stan sprawy
- **FAZA C — jakość i wygoda:**
  - reszta skilli, eskalacja, superwizorzy, re-identyfikacja, PDF, zestaw testowy

---

## 11. Mapa drogowa wersji

| Wersja | Sprzęt | Cel | Kryterium ukończenia |
|---|---|---|---|
| 0.x — Fundament | 16 GB | Zgody, zestaw testowy + wzorce, miernik jakości | Mam czym mierzyć i na czym testować |
| 1.0 — Działa end-to-end | 16 GB | Pełny przepływ; podział API/lokal ustalany testami | Sensowny raport; zmierzony recall anonimizatora; bazowy sweet spot |
| 2.0 — Lokalny dyrygent | 32 GB | Dyrygent (~14B) lokalnie; do API tylko wyspecjalizowane skille | Trafność dyrygowania w marginesie vs API |
| 3.0 — Migracja małych skilli | 32 GB+ | Jak najwięcej prostych skilli lokalnie | Każdy skill przeszedł "awans"; spadek wywołań API |
| 4.0 — W pełni lokalnie | 48 GB+ | Zero API; praca offline | Brak API; jakość w akceptowanej tolerancji (z sufitem) |

**Zasada "awansu" skilla:** skill przenosimy z API do lokalnego LLM dopiero, gdy lokalna wersja mieści się w ustalonym marginesie jakości względem wersji API.

---

## 12. Plan testowania

**Testy modułów osobno (każdy z input-fixture, miernikiem, kryterium):**
| Test | Moduł | Wejście testowe | Mierzymy | Zaliczenie |
|---|---|---|---|---|
| T1 | M1 | krótka próbka audio z iPhone | iCloud OFF, transfer bez chmury, format, jakość | nagranie OK, bez chmury |
| T2 | M2 | 20-min sesja (start: krótka próbka) | jakość polskiego, mówcy, czas, RAM | tekst czytelny, mówcy rozdzieleni |
| T3 | M3 | transkrypcja z **podłożonymi PII** | recall PII, fałszywe alarmy, flagowanie | **0 przepuszczonych** danych |
| T4a | M4a | fragment zanon + "wzorzec prawdy" | trafność każdego skilla | w marginesie vs wzorzec |
| T4b | M4b | sztuczny stan sprawy z konkurującymi hipotezami | trafność routingu + eskalacja | trafne decyzje |
| T4d | M4d | fragment z podłożonym sygnałem ryzyka | wykrywalność | 100% wykryć |
| T5 | M5 | raport z celowo wstawionymi błędami | wyłapywanie braków/błędów | łapie brak dowodów |
| T6 | M6 | raport + klucz | poprawność podstawień, czytelność PDF | dane przywrócone bezbłędnie |

**Strategia testów:** start od **1 transkrypcji wzorcowej ("książkowej")**, potem dokładamy kilka niezależnych, powtarzalnych testów. T2 startujemy od krótkiej próbki (2-3 min).

**Integracja (po zaliczeniu testów osobnych):**
1. I-A: M1+M2+M3 -> cała Faza A lokalna (audio -> zanon + klucz)
2. I-B: +M4 z jednym skillem -> pierwszy przebieg z API
3. I-C: +M4 pełne + M5 -> zespół + recenzja
4. I-D: +M6 -> pełny łańcuch do PDF

---

## 13. Słownik pojęć i ról

**Role / agenci**
- **Dyrygent** (= orkiestrator, agent główny, manager) — autonomiczny agent lokalny (OpenClaw). Prowadzi proces: czyta stan sprawy, wybiera i kolejkuje skille, decyduje o eskalacji do API, kończy pętlę.
- **Wyspecjalizowany skill** (mini-agent, specjalista) — wąski wykonawca jednego zadania wg kontraktu; rekomenduje kolejne kroki.
- **Superwizor** — skill recenzujący raport, najlepiej na innym modelu; szuka twierdzeń bez dowodów, braków, błędów.
- **Moduł ryzyka** — zawsze aktywny skill skanujący pod kątem ryzyka.
- **Pseudonimizator** — lokalny moduł zamieniający dane wrażliwe na zamienniki wg klucza.
- **Użytkownik** — terapeuta/student; zadaje polecenie, zatwierdza flagowane fragmenty, odbiera raport.

**Komponenty techniczne**
- **OpenClaw** — lokalny autonomiczny runtime agenta (kontroler + interfejs).
- **Bielik / model 7B** — lokalny LLM na Macu.
- **API** — zewnętrzny mocny model (tylko dane zanonimizowane).
- **WhisperX / whispermlx / MacWhisper** — lokalna transkrypcja audio -> tekst.
- **pyannote** — model rozpoznawania mówców (lokalnie).
- **Hugging Face** — platforma hostująca modele; źródło pyannote.

**Pojęcia procesu**
- **Granica prywatności** — krok 3 (pseudonimizacja), za który nie przechodzą dane wrażliwe.
- **Klucz** — odwracalny słownik mapowań (oryginał <-> zamiennik), w sejfie lokalnym.
- **Kontrakt skilla** — ustalony format wejścia/wyjścia (JSON).
- **Żywa konceptualizacja** (stan sprawy) — rosnący dokument roboczy; podstawa planowania dyrygenta.
- **Eskalacja** — decyzja dyrygenta o sięgnięciu po mocniejszy model (API).
- **Awans skilla** — przeniesienie skilla z API do lokalnego LLM przy zachowaniu jakości.

**Pojęcia testowe**
- **Fixture** — gotowy, ręczny input do testu modułu w izolacji.
- **Wzorzec / "książkowa" transkrypcja** — modelowa transkrypcja z podłożonymi PII i "prawdą" do oceny.
- **Recall PII** — odsetek wykrytych danych wrażliwych (cel: 100%).

---

## 14. Wizja na przyszłość (parking — nie rozwijać teraz)

- **Lokalność:** 2.0 lokalny dyrygent (14B/32GB), 3.0 migracja skilli, 4.0 pełna lokalność (48GB+); ewentualny fine-tuning Bielika.
- **Skala/pamięć:** "akta pacjenta" (RAG po wielu sesjach), batch kilkunastu transkrypcji, chunking + wyszukiwanie.
- **Jakość:** rozbudowa zestawu testowego, reguła awansu skilla, monitoring kosztu/czasu API.
- **Zespół agentów:** więcej skilli, grupa superwizorów (różne modele), dostrajanie reguł eskalacji.
- **Sterowanie kosztem:** dyrygent dostosowuje listę skilli do limitu tokenów (zwłaszcza przy API).
- **Interfejs/wyjście:** czat (Telegram) przez OpenClaw, szablony PDF, eksport do Word, prezentacja "żywej konceptualizacji".
- **Integracja transkrypcji:** skill w OpenClaw wywołujący lokalnie WhisperX (przed granicą prywatności, bez API).
- **Prywatność/prawo:** wybór dostawcy API + DPA + serwery UE, format odwracalnego klucza, DPIA.
- **Sprzęt audio:** mikrofon krawatowy, lepsza diaryzacja (pyannote) jeśli trzeba.
- **Kierunki otwarte:** treść kliniczna skilli do uźródłowienia (model CBT, listy kryteriów), ewentualna komercjalizacja.

---

## 15. Otwarte kwestie

- Dobór dostawcy API + DPA + lokalizacja serwerów (UE).
- Format/struktura klucza pseudonimizacji (kategorie zamienników, odwracalność lokalna).
- Reguły eskalacji do API (progi pewności, limit tokenów).
- Treść kliniczna skilli (który model konceptualizacji CBT, listy kryteriów) — obszar ekspercki użytkownika.
- Czy "żywa konceptualizacja" + RAG po wielu sesjach od razu, czy później.

---

## 16. Stan obecny i następny krok

- **Gdzie jesteśmy:** Faza A, przygotowanie testu **T2 (transkrypcja)**.
- **Decyzja narzędziowa:** start od `whispermlx` (Apple Silicon), fallback `whisperx`, plan B MacWhisper.
- **Najbliższy krok:** instalacja whispermlx + test T2 na krótkiej próbce (2-3 min); konto Hugging Face + token + akceptacja licencji pyannote (jednorazowo, do pobrania modelu mówców).
- **Potem:** T2 na pełnej 20-min sesji -> T3 (pseudonimizacja) na wyniku T2.
