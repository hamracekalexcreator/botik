# Krypto Discord Bot — Návod na spustenie

## Čo bot vie
- `/price BTC` — zobrazí aktuálnu cenu
- `/alert BTC 50000` — nastaví upozornenie, keď cena dosiahne 50 000 $
- `/myalerts` — zoznam tvojich aktívnych alertov
- `/removealert 3` — zmaže alert s daným ID
- `/analyzechart` — priložíš obrázok chartu, AI popíše viditeľné technické vzory (trend, support/resistance, formácie) — **nie je to predpoveď ceny**, len opis toho, čo je na obrázku vidieť

Bot kontroluje ceny každú minútu (cez CoinGecko API, zadarmo, bez potreby API kľúča).

---

## Krok 0: Nastavenie AI analýzy chartov (voliteľné, platené podľa použitia)

Ak chceš používať `/analyzechart`, potrebuješ Anthropic API kľúč:

1. Choď na https://console.anthropic.com a založ si účet
2. V sekcii **API Keys** vytvor nový kľúč
3. Dobi si kredit (stačí pár dolárov na start — jedna analýza obrázka stojí rádovo centy)
4. Skopírovaný kľúč pridáš do Railway Variables ako `ANTHROPIC_API_KEY` (rovnako ako `DISCORD_TOKEN`, pozri Krok 4 nižšie)

Ak tento kľúč nenastavíš, zvyšok bota (cenové alerty) funguje úplne normálne — len `/analyzechart` vypíše upozornenie, že chýba nastavenie.

---

## Krok 1: Vytvor Discord aplikáciu a bota

1. Choď na https://discord.com/developers/applications
2. Klikni **New Application**, pomenuj ju (napr. "Crypto Alert Bot")
3. V ľavom menu choď na **Bot** → **Add Bot**
4. Skopíruj si **Token** (klikni "Reset Token" ak treba) — toto je tvoj `DISCORD_TOKEN`
   ⚠️ Nikdy tento token nikde verejne nezdieľaj (napr. na GitHub)
5. V sekcii **Privileged Gateway Intents** nemusíš nič extra zapínať pre túto verziu
6. V ľavom menu choď na **OAuth2 → URL Generator**
   - Zaškrtni **bot** a **applications.commands**
   - V "Bot Permissions" zaškrtni: Send Messages, Read Message History, Use Slash Commands
   - Skopíruj vygenerovaný link a otvor ho v prehliadači — pridá bota na tvoj server

## Krok 2: Priprav si súbory (už máš)

Máš 3 súbory:
- `bot.py` — kód bota
- `requirements.txt` — zoznam knižníc
- `NAVOD.md` — tento návod

## Krok 3: Vyskúšaj lokálne (voliteľné, ak máš Python na počítači)

```bash
pip install -r requirements.txt
export DISCORD_TOKEN="tvoj_token_tu"     # na Windows: set DISCORD_TOKEN=tvoj_token_tu
python bot.py
```

Ak vidíš v termináli "Bot je online ako ...", funguje to. Choď na svoj Discord server a skús `/price BTC`.

## Krok 4: Nahraj na hosting, aby bežal 24/7 zadarmo

Odporúčam **Railway.app** (má bezplatný tier):

1. Založ si účet na https://railway.app (cez GitHub)
2. Vytvor nový repozitár na GitHube a nahraj tam tieto 3 súbory
3. V Railway: **New Project → Deploy from GitHub repo** → vyber svoj repozitár
4. V nastaveniach projektu (Variables) pridaj:
   - `DISCORD_TOKEN` = tvoj token z Kroku 1
5. Railway automaticky rozpozná Python projekt a spustí `python bot.py`
   (ak nie, nastav Start Command na `python bot.py`)
6. Po nasadení by mal bot naskočiť online na tvojom Discord serveri

**Alternatíva**: Render.com funguje podobne, tiež má free tier pre menšie projekty.

## Krok 5: Otestuj a zbieraj feedback

- Skús bota v 1-2 menších krypto Discord komunitách
- Sleduj, či niekto reálne používa `/alert` opakovane — to je signál, že má appka hodnotu
- Zapisuj si spätnú väzbu na to, čo by ľudia chceli navyše (napr. viac tokenov, rýchlejšie alerty)

---

## Poznámky a obmedzenia tejto verzie

- **Úložisko alertov**: momentálne sa ukladá do jednoduchého `alerts.json` súboru. Toto je v poriadku na začiatok, ale pri hostingu na Railway/Render sa súborový systém môže občas resetovať pri redeployi — pre produkčnú verziu by bolo lepšie použiť databázu (napr. Supabase, zadarmo tier).
- **CoinGecko free API** má rate limit (cca 10-30 requestov/minútu) — pri malom počte používateľov to stačí, pri väčšom raste treba zvážiť platený tier alebo cachovanie.
- Bot momentálne kontroluje ceny každých 60 sekúnd — to je nastaviteľné cez `CHECK_INTERVAL_SECONDS` v kóde.

## Ďalšie kroky (Fáza 2, keď bude fungovať základ)

- Pridať prudké pohyby ceny (napr. +/-5% za hodinu) bez nutnosti manuálne nastaviť alert
- Pridať news/sentiment monitoring (o čom sme sa bavili predtým)
- Pridať platenú rolu na serveri pre pokročilé funkcie
