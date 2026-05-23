---
name: market-analysis
description: >
  Cross-asset market analysis for trading decisions across DAX CFD, IBKR options, and crypto.
  Use when user asks for "market analysis", "what's the market doing", "macro context",
  "is it a good time to trade", "VIX level", "correlation indices crypto", "key levels",
  "support resistance", "upcoming macro events", "risk-on risk-off", "market regime".
  Provides structured analysis: macro events → indices → volatility → crypto → trading bias.
metadata:
  author: dnodal0
  version: 1.0.0
  assets: DAX/DE40, SPX, QQQ, BTC, ETH, crypto futures
  timeframes: intraday (15m) + swing (4h, daily)
---

# Market Analysis Framework

## Context

Tu trades simultanément 3 marchés distincts avec des logiques différentes :
- **DAX CFD** (IG Markets) : scalp/day trade 15min, SHORT dominant
- **IBKR Options** : Wheel strategy 30-45 DTE sur US equities/ETFs
- **Crypto Futures** : algo Freqtrade + positions discétionnaires sur Binance

L'analyse macro contextualise et priorise tes trades selon le régime de marché.

---

## Workflow d'analyse — Ordre d'exécution

```
1. Calendrier macro → 2. Indices majeurs → 3. Volatilité → 4. Crypto → 5. Bias final
```

---

## Étape 1 : Calendrier Macro (priorité absolue)

Vérifier les événements à fort impact dans les prochaines 48h :

| Événement | Impact DAX | Impact Options | Impact Crypto |
|-----------|-----------|----------------|---------------|
| BCE (taux/conférence) | 🔴 Très élevé | 🟡 Modéré | 🟡 Modéré |
| Fed (FOMC/minutes) | 🟡 Modéré | 🔴 Très élevé | 🔴 Très élevé |
| NFP (emploi US) | 🟡 Modéré | 🔴 Très élevé | 🟡 Modéré |
| CPI/PPI US | 🟡 Modéré | 🔴 Très élevé | 🔴 Très élevé |
| PIB Eurozone | 🔴 Très élevé | 🟢 Faible | 🟢 Faible |
| Earnings majeurs (AAPL, NVDA…) | 🟢 Faible | 🔴 Très élevé | 🟡 Modéré |

**RÈGLE** : Événement 🔴 dans les 30 minutes → suspendre les trades sur le marché impacté.

Sources à consulter : investing.com/economic-calendar, earnings whispers pour les earnings.

---

## Étape 2 : Analyse des Indices Majeurs

### DAX / DE40
- Niveau actuel vs EMA 20 (15min) et EMA 50 (15min)
- Position vs niveau psychologique le plus proche (arrondi)
- Structure : Higher Highs / Lower Lows → tendance
- Volume vs moyenne 20 périodes
- Spread bid/ask IG (acceptable < 1pt)

### S&P 500 / SPX et Nasdaq QQQ
- Corrélation avec DAX (généralement +0.75)
- Si SPX en fort mouvement → anticiper contagion DAX dans 30-60min
- SPX au-dessus/en dessous de la MA200 journalière → régime bull/bear

### Régime de marché

| Condition | Régime | Implication |
|-----------|--------|-------------|
| SPX > MA200 + VIX < 20 | 🟢 Risk-On | CSP éloignés, Wheel favorable, crypto long |
| SPX < MA200 + VIX > 25 | 🔴 Risk-Off | CSP plus proches, pas de Wheel, crypto neutre/short |
| VIX 20-25 | 🟡 Transition | Réduire taille, DTE plus courts |
| VIX > 30 | ⚫ Crise | Suspendre Wheel, DAX scalp seulement |

---

## Étape 3 : Volatilité

### VIX (Options US)
```
VIX < 15  → IV Rank généralement bas → difficile de vendre du premium
VIX 15-20 → Zone neutre
VIX 20-25 → Bon contexte pour vendre des CSPs (IV Rank > 30 probable)
VIX > 25  → Premium élevé MAIS risque de gap important
VIX > 30  → Uniquement stratégies défensives
```

### VDAX (Options DAX)
- Pendant sessions européennes, surveiller VDAX pour contexte DAX
- VDAX > 25 → élargissement des spreads IG, slippage potentiel

### IV Rank implicite
En l'absence d'IV Rank exact, estimer via VIX :
- VIX au 52w high → IV Rank ≈ 80-100%
- VIX au 52w low → IV Rank ≈ 0-20%
- VIX médiane → IV Rank ≈ 40-60%

---

## Étape 4 : Crypto Market Structure

### BTC Dominance
```
BTC Dominance > 55% → Risk-off crypto, privilégier BTC sur altcoins
BTC Dominance < 45% → Altseason possible, diversification acceptable
```

### Funding Rate (Binance Futures)
```
Funding Rate > +0.1%  → Longs surreprésentés → risque de long squeeze
Funding Rate < -0.1%  → Shorts surreprésentés → risque de short squeeze
Funding Rate ≈ 0      → Marché équilibré, plus sain
```

### Corrélations cross-asset
- BTC corrèle avec QQQ (Nasdaq) dans les phases de stress : corr ≈ +0.6 à +0.8
- DAX corrèle avec SPX : corr ≈ +0.7
- En Risk-Off : BTC chute souvent avant les indices

---

## Étape 5 : Bias de Trading Final

Produire un résumé actionnable par marché :

```
╔══════════════════════════════════════════════════╗
║           MARKET ANALYSIS — [DATE/HEURE]         ║
╠══════════════════════════════════════════════════╣
║ MACRO     : [Événements à risque aujourd'hui]    ║
║ RÉGIME    : 🟢 Risk-On / 🟡 Neutre / 🔴 Risk-Off ║
║ VIX       : XX.X  [commentaire]                  ║
╠══════════════════════════════════════════════════╣
║ DAX/DE40                                         ║
║   Niveau  : XXXXX  Tendance: ↑/↓/→              ║
║   EMA20   : XXXXX  (prix au-dessus/dessous)      ║
║   Bias    : LONG / SHORT / NEUTRE                ║
║   Alerte  : [macro dans XX min ?]                ║
╠══════════════════════════════════════════════════╣
║ OPTIONS IBKR                                     ║
║   IV Rank contexte : ✅ favorable / ❌ défavorable ║
║   Wheel   : GO / NO-GO                           ║
║   CSP cible : [ticker si applicable]             ║
╠══════════════════════════════════════════════════╣
║ CRYPTO FUTURES                                   ║
║   BTC     : $XXXXX  Tendance: ↑/↓/→             ║
║   Funding : +X.XXX%  [commentaire]               ║
║   Freqtrade : [actif / surveiller / pause]       ║
║   Opportunité : [paire si applicable]            ║
╠══════════════════════════════════════════════════╣
║ PRIORITÉ DU JOUR                                 ║
║ 1. [action prioritaire]                          ║
║ 2. [action secondaire]                           ║
║ 3. [à surveiller]                                ║
╚══════════════════════════════════════════════════╝
```

---

## Règles cross-marché

### Corrélations à respecter
- Si DAX en forte tendance baissière → ne pas ouvrir de LONG crypto
- Si VIX spike > +20% en une journée → suspendre toutes nouvelles positions
- Si USD/EUR bouge > 0.5% → impact probable sur DAX et actions US

### Capital allocation par régime

| Régime | DAX CFD | IBKR Wheel | Crypto Futures |
|--------|---------|-----------|----------------|
| 🟢 Risk-On  | 30% | 50% | 20% |
| 🟡 Neutre   | 40% | 40% | 20% |
| 🔴 Risk-Off | 60% | 20% | 20% (hedges) |
| ⚫ Crise    | 80% (scalp only) | 0% | 20% (BTC only) |

---

## Sources d'information rapides

| Info | Source |
|------|--------|
| Calendrier macro | investing.com/economic-calendar |
| VIX en temps réel | finance.yahoo.com/quote/%5EVIX |
| Funding rates crypto | binance.com/en/futures (ou coinglass.com) |
| BTC Dominance | coinmarketcap.com |
| IV Options US | marketchameleon.com ou IBKR Trader Workstation |
| Niveaux DAX | IG Markets plateforme + ProRealTime |
