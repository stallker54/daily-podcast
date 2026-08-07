# Investování s rozhledem — daily investing podcast

Automaticky generovaný denní podcast o dlouhodobém investování (akcie,
nemovitosti, krypto a další aktiva) s příležitostným krátkodobým tipem.

Postaveno podle:

```text
Claude Code Routine -> live web research -> input/narration.txt
GitHub Actions      -> OpenAI text-to-speech -> MP3 -> GitHub Release
GitHub Pages        -> docs/feed.xml -> RSS
Podcast app          -> přehrávání
```

Kompletní návod na nastavení je v přiloženém dokumentu
`setup-guide-investovani.md`.

## Struktura repozitáře

```text
.github/workflows/publish-podcast.yml   GitHub Actions workflow
scripts/generate_podcast.py             TTS, kontrola délky, RSS
claude-routine-instructions-example.md  Prompt pro Claude Routine
input/narration.txt                     Výstup Routine, spouští workflow
state/reported-stories.json             Historie nahlášených zpráv (deduplikace)
docs/                                   Veřejný feed a landing page (GitHub Pages)
```

## Bezpečnost

Repozitář je veřejný. `OPENAI_API_KEY` je uložen výhradně jako GitHub Actions
secret a nikdy se neobjevuje v promptu ani v Claude cloud prostředí.
