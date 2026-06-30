# Dziennik decyzji projektu

> Trwały zapis ustalonych decyzji. Najnowsze na górze.

## Transkrypcja — wybór narzędzia
**Decyzja:** Próbujemy zintegrować z projektem **WhisperX** jako narzędzie do transkrypcji.
Jeśli będzie działał słabo (jakość polskiego, rozdzielanie mówców, szybkość na Mac M4) —
**plan B: MacWhisper**.

**Uzasadnienie:**
- WhisperX jest darmowy i obejmuje rozdzielanie mówców (diarization).
- Jest skryptowalny (CLI), więc pasuje do docelowej automatyzacji.
- MacWhisper jako fallback: łatwy, szybszy na Apple Silicon, ale GUI (trudny do automatyzacji), diarization płatna (Pro).

**Kolejność działania:**
1. Najpierw test WhisperX samodzielnie (T2) — sprawdzić jakość polskiego i rozdzielanie mówców na M4.
2. Jeśli dobrze → integracja jako lokalny skill w OpenClaw (patrz parking poniżej).
3. Jeśli słabo → przejście na MacWhisper.

## Parking pomysłów (punkt E) — dopisek
- **Skill transkrypcji w OpenClaw wywołujący lokalnie WhisperX** — przed granicą prywatności,
  w 100% lokalnie, BEZ API (audio nigdy nie opuszcza Maca). OpenClaw uruchamia WhisperX przez `exec`.

## Warunek wstępny (bezpieczeństwo)
- Przed jakąkolwiek instalacją: Mac musi być bezpieczny po incydencie z fałszywą stroną
  (`macwhisper.org` / `filetilapiahub.com` → złośliwy skrypt `curl | zsh`).
  Instalacje tylko z zaufanych źródeł (App Store / oficjalne PyPI / GitHub).
