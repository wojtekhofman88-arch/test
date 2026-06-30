# PROTOKOŁY OPERACYJNE — jak wykonać poszczególne kroki

> Uzupełnienie do `RAPORT.md`. Raport mówi CO i DLACZEGO; ten plik mówi JAK.
> (Wątek incydentu bezpieczeństwa pominięty — sprawa zamknięta.)

---

## 1. Protokół T1 — pozyskanie audio (iPhone)

**Cel:** nagrać sesję iPhonem i przenieść na Maca BEZ chmury, w jakości nadającej się do transkrypcji.

**Kroki:**
1. **Zgoda pacjenta** na nagrywanie (warunek konieczny). Na testy: sesja odegrana/fikcyjna.
2. **Wyłącz iCloud dla Dyktafonu:** Ustawienia → [Twoje imię] → iCloud → "Pokaż wszystko" → Dyktafon → wyłącz. (Alternatywa: nagrywaj w trybie samolotowym.)
3. Nagraj w **cichym pomieszczeniu**, telefon na środku między rozmówcami (~50-80 cm od każdego).
4. Aplikacja **Dyktafon** → nagraj.
5. **Transfer na Maca:** kabel USB-C lub AirDrop. Zapisz np. w `~/transkrypcje/`.
6. **Skasuj nagranie z telefonu** po transferze.

**Karta oceny T1:**
| Co sprawdzamy | Kryterium zaliczenia |
|---|---|
| iCloud dla Dyktafonu wyłączony | nagranie nie synchronizuje się do chmury |
| Nagranie udane | plik istnieje, da się odtworzyć |
| Transfer bez chmury | przeszło kablem/AirDrop |
| Format pliku | `.m4a` lub `.wav` |
| Jakość audio | oba głosy słyszalne i wyraźne |
| Kasowanie z telefonu | usunięte po transferze |

---

## 2. Protokół T2 — instalacja whispermlx + test transkrypcji

**Cel:** sprawdzić jakość polskiej transkrypcji i rozdzielania mówców na Macu M4.
**Uwaga:** instaluj tylko z zaufanych źródeł (Homebrew/PyPI). Idź krok po kroku.

### Krok 1 — Homebrew (jeśli nie ma)
W Terminalu, oficjalna komenda ze strony brew.sh:
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Krok 2 — Python i ffmpeg
```
brew install python@3.11 ffmpeg
```

### Krok 3 — osobne środowisko Pythona
```
python3.11 -m venv ~/whisper-env
source ~/whisper-env/bin/activate
```

### Krok 4 — instalacja whispermlx
```
pip install whispermlx
```
(Fallback, jeśli whispermlx zawiedzie: `pip install whisperx` — wolniejszy na Macu, bo liczy na CPU.)

### Krok 5 — token Hugging Face (do rozdzielania mówców) — patrz sekcja 3.

### Krok 6 — przygotuj próbkę
Wrzuć nagranie (2-3 min na start) np. do `~/transkrypcje/test.m4a`.

### Krok 7 — uruchom transkrypcję
```
whispermlx ~/transkrypcje/test.m4a --model large-v3 --language pl --diarize --hf_token TWOJ_TOKEN --output_dir ~/transkrypcje
```
(`--diarize` = rozdzielanie mówców; `--language pl` = polski; `--model large-v3` — można też `large-v3-turbo` dla szybkości.)

**Karta oceny T2:**
| Co oceniamy | Kryterium |
|---|---|
| Instalacja | bez błędów |
| Jakość polskiego | tekst zrozumiały, błędy nieliczne |
| Rozdzielanie mówców | dwa głosy rozróżnione poprawnie |
| Czas | akceptowalny (zmierz stoperem) |
| RAM (Monitor aktywności) | bez zapychania/swapowania |

**Dostrajanie wg wyników:**
- słaba jakość → model `large-v3` zamiast turbo, lepsze audio (mikrofon, cisza)
- źle rozdzieleni mówcy → zmiana ustawień diaryzacji lub fallback whisperx/MacWhisper
- za wolno / RAM na styk → model turbo, krótsze segmenty

---

## 3. Instrukcja Hugging Face (konto + licencja + token)

Potrzebne TYLKO do pobrania modelu rozpoznawania mówców (pyannote). **Audio nigdzie nie jest wysyłane** — po pobraniu diaryzacja działa lokalnie.

**Kroki (jednorazowo):**
1. Załóż darmowe konto na **huggingface.co**.
2. Wejdź na stronę modelu **`pyannote/speaker-diarization-3.1`** → **zaakceptuj warunki** (czasem krótki formularz).
3. To samo dla **`pyannote/segmentation-3.0`**.
4. Ustawienia konta → **Access Tokens** → utwórz token typu "read" → skopiuj.
5. Token wklejasz w komendzie z Kroku 7 (parametr `--hf_token`).

**Co to znaczy (w skrócie):**
- **Konto** = Twoja tożsamość na platformie modeli.
- **Akceptacja licencji** = zgoda autora pyannote na użycie modelu ("gated model").
- **Token** = klucz dostępu, dzięki któremu program sam pobierze model bez podawania hasła.

---

## 4. Ranking narzędzi transkrypcji (uzasadnienie decyzji)

**Wymagania:** plik nagrania rozmowy 2 osób + rozdzielanie mówców + lokalnie + polski + sensowna cena.

**WAŻNE rozróżnienie:** transkrypcja pliku z mówcami ≠ dyktando (voice typing).
Aplikacje-dyktanda (Superwhisper, VoiceInk, Voibe, Wispr Flow) NIE rozdzielają mówców z pliku — dlatego odpadają.

**Pasujące pod nasze wymagania:**
| # | Narzędzie | Cena | Mówcy | Lokalnie | Łatwość |
|---|---|---|---|---|---|
| 1 | MacWhisper (Pro) | ~€29-79 jednorazowo (jest darmowy tier) | tak (Pro) | tak | bardzo łatwa (GUI) |
| 2 | whispermlx (MLX/Apple) | darmowe | tak | tak | trudna (CLI), ale szybka na M4 |
| 3 | WhisperX | darmowe | tak | tak | trudna (CLI), CPU na Macu = wolniej |
| 4 | Buzz | darmowe | ograniczone | tak | łatwa (GUI) |
| 5 | whisper.cpp / Whisper | darmowe | brak | tak | trudna |

**Decyzja:** whispermlx (1. wybór na M4, bo GPU/MLX + skryptowalny) → whisperx (fallback) → MacWhisper (plan B, łatwy ale GUI/płatny diarization).
Powód wyboru rodziny WhisperX nad MacWhisper: **skryptowalność** (potrzebna do automatyzacji w OpenClaw).

**Model:** Whisper `large-v3` / `large-v3-turbo` (turbo lżejszy, mieści się w 16 GB).

**Bezpieczeństwo instalacji:** tylko App Store / oficjalne PyPI / GitHub / ręcznie wpisany `macwhisper.com`. Nigdy z linków z wyszukiwarki/reklam.

---

## 5. Alternatywy dla OpenClaw (rozważane)

| Narzędzie | Charakter | Dla nas |
|---|---|---|
| **OpenClaw** | autonomiczny lokalny agent, skille (SKILL.md), interfejs czat | **wybrane** — pasuje do wizji dyrygenta |
| LangGraph / CrewAI / AutoGen | frameworki agentowe | wymagają programowania — za trudne na teraz |
| n8n / Flowise | wizualna automatyzacja (bloki) | przystępniejsze niż kod, opcja zapasowa |
| Dify | self-hosted platforma aplikacji LLM z UI | sporo bez kodu, opcja zapasowa |
| Open WebUI | ładny czat na Ollama | proste, ale to nie orkiestrator |

**Uwaga krytyczna:** dla samego liniowego przepływu OpenClaw bywa przerostem; jego sens to dynamiczny rdzeń rozumowania (dyrygowanie zespołem skilli). Decyzję o ostatecznej architekturze orkiestracji utrzymujemy, ale w Fazie A agent nie jest jeszcze potrzebny.
