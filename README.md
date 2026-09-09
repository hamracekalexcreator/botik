[bot (1).py](https://github.com/user-attachments/files/32014783/bot.1.py)
"""
Krypto Discord Bot - Cenové alerty
-----------------------------------
Funkcie:
- /price <token>  -> aktuálna cena tokenu
- /alert <token> <cena>  -> upozorní, keď token dosiahne danú cenu
- /myalerts  -> zobrazí tvoje aktívne alerty
- /removealert <id>  -> zmaže konkrétny alert

Dáta: CoinGecko API (zadarmo, bez potreby API kľúča pre základné použitie)
"""

import discord
from discord import app_commands
from discord.ext import commands, tasks
import aiohttp
import json
import os
from datetime import datetime

# ---------------------------------------------------------
# KONFIGURÁCIA
# ---------------------------------------------------------
DISCORD_TOKEN = os.environ.get("DISCORD_TOKEN")  # nastavíš v hostingu (Railway/Render) ako env variable
COINGECKO_API = "https://api.coingecko.com/api/v3"
ALERTS_FILE = "alerts.json"
CHECK_INTERVAL_SECONDS = 60  # ako často bot kontroluje ceny (1 minúta)

# Mapovanie najbežnejších skratiek na CoinGecko ID — slúži ako rýchla skratka
# (bez volania API) a hlavne na vyriešenie kolízií, keď search vráti nesprávny
# výsledok (napr. "ton" by sa mohlo nájsť ako menej známy token namiesto Toncoinu).
TOKEN_ID_MAP = {
    "btc": "bitcoin",
    "eth": "ethereum",
    "sol": "solana",
    "bnb": "binancecoin",
    "xrp": "ripple",
    "doge": "dogecoin",
    "ada": "cardano",
    "avax": "avalanche-2",
    "dot": "polkadot",
    "matic": "matic-network",
    "link": "chainlink",
    "ton": "the-open-network",
    "trx": "tron",
    "shib": "shiba-inu",
}

# Cache pre výsledky vyhľadávania, aby sme pri opakovanom dopyte na rovnaký
# token nemuseli znova volať CoinGecko search (šetrí rate limit).
# Formát: {"pepe": "pepe", "neznamy_token": None, ...}
_search_cache: dict[str, str | None] = {}

# ---------------------------------------------------------
# NASTAVENIE BOTA
# ---------------------------------------------------------
intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)


def load_alerts():
    """Načíta uložené alerty zo súboru (jednoduché JSON úložisko)."""
    if not os.path.exists(ALERTS_FILE):
        return []
    with open(ALERTS_FILE, "r") as f:
        return json.load(f)


def save_alerts(alerts):
    """Uloží alerty do súboru."""
    with open(ALERTS_FILE, "w") as f:
        json.dump(alerts, f, indent=2)


async def resolve_token_id(session: aiohttp.ClientSession, token: str):
    """
    Prevedie ticker/názov (napr. 'btc' alebo 'pepe') na presné CoinGecko id
    (napr. 'bitcoin' alebo 'pepe').

    Poradie hľadania:
    1. Ručná mapa (najrýchlejšie, žiadne API volanie) — pre najbežnejšie tokeny
    2. Cache z predošlých úspešných vyhľadávaní
    3. CoinGecko /search endpoint — nájde takmer akýkoľvek existujúci token

    Vráti dvojicu (token_id, suggestions):
    - Ak sa našla istá zhoda: (token_id, [])
    - Ak sa nenašla istá zhoda, ale existujú podobné tokeny: (None, [zoznam návrhov])
    - Ak sa nenašlo vôbec nič: (None, [])
    """
    token = token.lower().strip()

    if token in TOKEN_ID_MAP:
        return TOKEN_ID_MAP[token], []

    if token in _search_cache:
        return _search_cache[token], []

    token_id, suggestions = await _search_token(session, token)
    if token_id is not None:
        _search_cache[token] = token_id
    return token_id, suggestions


async def _search_token(session: aiohttp.ClientSession, query: str):
    """
    Zavolá CoinGecko /search. Ak nájde token, ktorého symbol alebo názov
    presne sedí so zadaným textom, vráti ho ako istú zhodu. Ak presná zhoda
    neexistuje, ale search vrátil podobné výsledky, vráti ich ako návrhy
    (namiesto toho, aby si potichu vybral len prvý výsledok — to by mohlo
    trafiť úplne iný token, než aký používateľ myslel).

    Vráti dvojicu (token_id_alebo_None, suggestions).
    """
    url = f"{COINGECKO_API}/search"
    params = {"query": query}
    async with session.get(url, params=params) as resp:
        if resp.status != 200:
            return None, []
        data = await resp.json()

    coins = data.get("coins", [])
    if not coins:
        return None, []

    # 1. priorita: presná zhoda symbolu (ticker)
    for coin in coins:
        if coin.get("symbol", "").lower() == query:
            return coin["id"], []

    # 2. priorita: presná zhoda názvu
    for coin in coins:
        if coin.get("name", "").lower() == query:
            return coin["id"], []

    # 3. žiadna istá zhoda — ponúkni najbližšie 3 výsledky ako návrhy na výber
    suggestions = [
        {
            "id": c["id"],
            "name": c.get("name", "?"),
            "symbol": c.get("symbol", "?").upper(),
        }
        for c in coins[:3]
    ]
    return None, suggestions


async def fetch_price(session: aiohttp.ClientSession, token_id: str):
    """Zavolá CoinGecko API a vráti aktuálnu cenu v USD, alebo None ak zlyhá."""
    url = f"{COINGECKO_API}/simple/price"
    params = {"ids": token_id, "vs_currencies": "usd"}
    async with session.get(url, params=params) as resp:
        if resp.status != 200:
            return None
        data = await resp.json()
        if token_id not in data:
            return None
        return data[token_id]["usd"]


def format_price(value: float) -> str:
    """
    Naformátuje cenu tak, aby boli vidieť aj veľmi lacné tokeny (napr. SHIB,
    PEPE), ktoré majú cenu rádovo v milióntinách centu. Pri bežných cenách
    (nad 1 cent) použije klasické 2-4 desatinné miesta; pri veľmi malých
    hodnotách zobrazí toľko desatinných miest, aby boli vidno prvé platné
    číslice (namiesto zaokrúhlenia na 0.0000).
    """
    if value == 0:
        return "0.00"
    if value >= 1:
        return f"{value:,.4f}"
    if value >= 0.01:
        return f"{value:,.6f}"

    # veľmi malá hodnota (napr. 0.00000876) — zisti, koľko núl je za desatinnou
    # čiarkou pred prvou platnou číslicou, a zobraz 4 platné číslice navyše
    exponent = 0
    temp = value
    while temp < 0.1:
        temp *= 10
        exponent += 1
    decimals = exponent + 4
    return f"{value:.{decimals}f}"


def format_suggestions(suggestions: list) -> str:
    """Naformátuje zoznam návrhov tokenov do prehľadného zoznamu pre Discord správu."""
    lines = [f"• `{s['symbol']}` — {s['name']}" for s in suggestions]
    return "\n".join(lines)


# ---------------------------------------------------------
# PRÍKAZY (slash commands)
# ---------------------------------------------------------

@bot.event
async def on_ready():
    await bot.tree.sync()
    print(f"Bot je online ako {bot.user}")
    if not check_alerts_loop.is_running():
        check_alerts_loop.start()


@bot.tree.command(name="price", description="Zobrazí aktuálnu cenu kryptomeny")
@app_commands.describe(token="Skratka tokenu, napr. BTC alebo ETH")
async def price(interaction: discord.Interaction, token: str):
    await interaction.response.defer()

    async with aiohttp.ClientSession() as session:
        token_id, suggestions = await resolve_token_id(session, token)
        if token_id is None:
            if suggestions:
                await interaction.followup.send(
                    f"❌ Nenašiel som presnú zhodu pre `{token}`. Možno si myslel(a):\n"
                    f"{format_suggestions(suggestions)}\n\n"
                    f"Skús znova s presným tickerom alebo názvom z tohto zoznamu."
                )
            else:
                await interaction.followup.send(
                    f"❌ Nenašiel som žiadny token pre `{token}`. Skontroluj názov a skús znova."
                )
            return
        current_price = await fetch_price(session, token_id)

    if current_price is None:
        await interaction.followup.send(
            f"❌ Token `{token}` sa našiel, ale nepodarilo sa načítať jeho cenu. Skús znova."
        )
        return

    await interaction.followup.send(
        f"💰 **{token.upper()}**: ${format_price(current_price)}"
    )


@bot.tree.command(name="alert", description="Nastaví cenový alert pre token")
@app_commands.describe(token="Skratka tokenu, napr. BTC", cena="Cieľová cena v USD")
async def alert(interaction: discord.Interaction, token: str, cena: float):
    await interaction.response.defer()

    # over, že token vôbec existuje, predtým než uložíme alert
    async with aiohttp.ClientSession() as session:
        token_id, suggestions = await resolve_token_id(session, token)
        if token_id is None:
            if suggestions:
                await interaction.followup.send(
                    f"❌ Nenašiel som presnú zhodu pre `{token}`. Možno si myslel(a):\n"
                    f"{format_suggestions(suggestions)}\n\n"
                    f"Skús znova s presným tickerom alebo názvom z tohto zoznamu."
                )
            else:
                await interaction.followup.send(
                    f"❌ Nenašiel som žiadny token pre `{token}`. Skontroluj názov a skús znova."
                )
            return
        current_price = await fetch_price(session, token_id)

    if current_price is None:
        await interaction.followup.send(
            f"❌ Token `{token}` sa našiel, ale nepodarilo sa načítať jeho cenu. Skús znova."
        )
        return

    alerts = load_alerts()
    new_alert = {
        "id": len(alerts) + 1,
        "user_id": interaction.user.id,
        "channel_id": interaction.channel_id,
        "token_id": token_id,
        "token_label": token.upper(),
        "target_price": cena,
        "direction": "above" if cena > current_price else "below",
        "created_at": datetime.utcnow().isoformat(),
        "triggered": False,
    }
    alerts.append(new_alert)
    save_alerts(alerts)

    smer = "vzrastie nad" if new_alert["direction"] == "above" else "klesne pod"
    await interaction.followup.send(
        f"✅ Alert nastavený! Upozorním ťa, keď **{token.upper()}** {smer} **${cena:,.6f}**\n"
        f"(aktuálna cena: ${format_price(current_price)})"
    )


@bot.tree.command(name="myalerts", description="Zobrazí tvoje aktívne alerty")
async def myalerts(interaction: discord.Interaction):
    alerts = load_alerts()
    user_alerts = [
        a for a in alerts
        if a["user_id"] == interaction.user.id and not a["triggered"]
    ]

    if not user_alerts:
        await interaction.response.send_message("Nemáš žiadne aktívne alerty.")
        return

    lines = []
    for a in user_alerts:
        smer = "↑" if a["direction"] == "above" else "↓"
        lines.append(f"`#{a['id']}` {a['token_label']} {smer} ${format_price(a['target_price'])}")

    await interaction.response.send_message("📋 **Tvoje alerty:**\n" + "\n".join(lines))


@bot.tree.command(name="removealert", description="Zmaže konkrétny alert podľa ID")
@app_commands.describe(alert_id="ID alertu (zobrazíš cez /myalerts)")
async def removealert(interaction: discord.Interaction, alert_id: int):
    alerts = load_alerts()
    before = len(alerts)
    alerts = [
        a for a in alerts
        if not (a["id"] == alert_id and a["user_id"] == interaction.user.id)
    ]

    if len(alerts) == before:
        await interaction.response.send_message("❌ Alert s týmto ID sa nenašiel (alebo nie je tvoj).")
        return

    save_alerts(alerts)
    await interaction.response.send_message(f"🗑️ Alert `#{alert_id}` bol zmazaný.")


# ---------------------------------------------------------
# BACKGROUND ÚLOHA - kontrola alertov
# ---------------------------------------------------------

@tasks.loop(seconds=CHECK_INTERVAL_SECONDS)
async def check_alerts_loop():
    """Pravidelne kontroluje ceny a spúšťa notifikácie, keď je podmienka splnená."""
    alerts = load_alerts()
    active_alerts = [a for a in alerts if not a["triggered"]]

    if not active_alerts:
        return

    # zoskup podľa token_id, aby sme nevolali API zbytočne veľakrát pre rovnaký token
    unique_token_ids = list(set(a["token_id"] for a in active_alerts))

    async with aiohttp.ClientSession() as session:
        prices = {}
        for token_id in unique_token_ids:
            p = await fetch_price(session, token_id)
            if p is not None:
                prices[token_id] = p

    changed = False
    for a in active_alerts:
        current = prices.get(a["token_id"])
        if current is None:
            continue

        hit = (
            (a["direction"] == "above" and current >= a["target_price"]) or
            (a["direction"] == "below" and current <= a["target_price"])
        )

        if hit:
            channel = bot.get_channel(a["channel_id"])
            if channel:
                await channel.send(
                    f"🚨 <@{a['user_id']}> **{a['token_label']}** dosiahol tvoju cieľovú cenu!\n"
                    f"Aktuálna cena: **${format_price(current)}** (cieľ: ${format_price(a['target_price'])})"
                )
            a["triggered"] = True
            changed = True

    if changed:
        save_alerts(alerts)


# ---------------------------------------------------------
# ŠTART BOTA
# ---------------------------------------------------------

if __name__ == "__main__":
    if not DISCORD_TOKEN:
        raise RuntimeError(
            "Chýba DISCORD_TOKEN. Nastav ho ako environment variable pred spustením."
        )
    bot.run(DISCORD_TOKEN)
