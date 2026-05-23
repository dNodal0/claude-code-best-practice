---
name: freqtrade-strategy-dev
description: >
  Freqtrade strategy development, backtesting, hyperopt, and live trading on Binance Futures.
  Use when user asks to "create a freqtrade strategy", "backtest", "hyperopt", "optimize parameters",
  "freqtrade config", "dry-run", "live trading crypto", "Binance futures bot",
  or mentions ROI table, stoploss, trailing stop, buy/sell signals in Freqtrade context.
  Follows dnodal0's workflow: strategy → backtest → hyperopt → dry-run → live.
metadata:
  author: dnodal0
  version: 1.0.0
  framework: Freqtrade
  exchange: Binance Futures (USDT perpetual)
  arch: ARM64 / Docker Compose
---

# Freqtrade Strategy Developer

## Context

Tu développes des stratégies algo crypto avec Freqtrade, déployé en Docker Compose sur ARM64 (M-series Mac / Raspberry Pi).
Exchange principal : Binance Futures (contrats USDT perpetual).
Stack MCP : `mcp__freqtrade-mcp__*` disponible pour interagir avec le bot en live.

---

## Workflow de développement — Pipeline standard

```
1. Conception strategy → 2. Backtest → 3. Hyperopt → 4. Dry-run → 5. Live (small size)
```

Ne jamais sauter d'étape. Ne jamais passer en live sans dry-run validé ≥ 7 jours.

---

## Étape 1 : Structure d'une stratégie Freqtrade

### Template de base

```python
from freqtrade.strategy import IStrategy, IntParameter, DecimalParameter
from pandas import DataFrame
import talib.abstract as ta

class MyStrategy(IStrategy):
    # ─── Risk Management ───────────────────────────────────────
    minimal_roi = {
        "0":   0.03,   # 3% immédiatement
        "30":  0.02,   # 2% après 30min
        "60":  0.01,   # 1% après 1h
        "120": 0      # Break-even après 2h
    }
    stoploss = -0.02           # -2% hard stop
    trailing_stop = True
    trailing_stop_positive = 0.01
    trailing_stop_positive_offset = 0.015
    trailing_only_offset_is_reached = True

    # ─── Timeframe ─────────────────────────────────────────────
    timeframe = '15m'
    use_exit_signal = True
    exit_profit_only = False
    ignore_roi_if_entry_signal = False

    # ─── Hyperopt Parameters ───────────────────────────────────
    rsi_buy = IntParameter(20, 40, default=30, space='buy')
    rsi_sell = IntParameter(60, 80, default=70, space='sell')
    ema_fast = IntParameter(8, 21, default=12, space='buy')
    ema_slow = IntParameter(20, 50, default=26, space='buy')

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        dataframe['ema_fast'] = ta.EMA(dataframe, timeperiod=self.ema_fast.value)
        dataframe['ema_slow'] = ta.EMA(dataframe, timeperiod=self.ema_slow.value)
        dataframe['volume_mean'] = dataframe['volume'].rolling(20).mean()
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (dataframe['rsi'] < self.rsi_buy.value) &
            (dataframe['ema_fast'] > dataframe['ema_slow']) &
            (dataframe['volume'] > dataframe['volume_mean']) &
            (dataframe['close'] > dataframe['close'].shift(1)),
            'enter_long'
        ] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (dataframe['rsi'] > self.rsi_sell.value) |
            (dataframe['ema_fast'] < dataframe['ema_slow']),
            'exit_long'
        ] = 1
        return dataframe
```

---

## Étape 2 : Backtest

### Commandes Docker

```bash
# Télécharger les données historiques
docker compose run --rm freqtrade download-data \
  --exchange binance \
  --pairs BTC/USDT ETH/USDT SOL/USDT \
  --timeframes 15m 1h \
  --days 365

# Lancer le backtest
docker compose run --rm freqtrade backtesting \
  --strategy MyStrategy \
  --timerange 20230101-20240101 \
  --timeframe 15m \
  --export trades \
  --export-filename user_data/backtest_results/my_strategy.json
```

### Métriques à vérifier (seuils minimaux)

| Métrique | Seuil minimal | Seuil bon |
|----------|--------------|-----------|
| Profit total | > 0% | > 20%/an |
| Win rate | > 45% | > 55% |
| Profit Factor | > 1.2 | > 1.5 |
| Max Drawdown | < 20% | < 10% |
| Sharpe Ratio | > 0.5 | > 1.0 |
| Nb trades | > 50 | > 100 |
| Avg trade duration | cohérent avec timeframe | — |

**RÈGLE** : Si Max Drawdown > 25% → rejeter la stratégie sans hyperopt.

---

## Étape 3 : Hyperopt

```bash
# Optimisation des paramètres d'entrée
docker compose run --rm freqtrade hyperopt \
  --strategy MyStrategy \
  --hyperopt-loss SharpeHyperOptLoss \
  --spaces buy sell roi stoploss \
  --epochs 200 \
  --timerange 20230101-20231001 \
  --jobs -1

# Appliquer les meilleurs paramètres
docker compose run --rm freqtrade hyperopt-show \
  --best \
  --no-header
```

### Fonctions de loss disponibles

| Loss Function | Quand l'utiliser |
|---------------|-----------------|
| `SharpeHyperOptLoss` | Default — bon équilibre risque/rendement |
| `SortinoHyperOptLoss` | Si tu veux minimiser les drawdowns |
| `MaxDrawDownHyperOptLoss` | Focus protection capital |
| `CalmarHyperOptLoss` | Ratio rendement/drawdown max |

**RÈGLE** : Valider les paramètres hyperoptimisés sur une période out-of-sample
(données non utilisées pendant l'hyperopt) avant de passer en dry-run.

---

## Étape 4 : Configuration live / dry-run

### config.json — Template Binance Futures

```json
{
  "max_open_trades": 3,
  "stake_currency": "USDT",
  "stake_amount": "unlimited",
  "tradable_balance_ratio": 0.95,
  "fiat_display_currency": "EUR",
  "timeframe": "15m",
  "dry_run": true,
  "dry_run_wallet": 1000,
  "cancel_open_orders_on_exit": false,

  "exchange": {
    "name": "binance",
    "key": "${BINANCE_API_KEY}",
    "secret": "${BINANCE_API_SECRET}",
    "ccxt_config": {
      "defaultType": "future"
    },
    "pair_whitelist": ["BTC/USDT:USDT", "ETH/USDT:USDT", "SOL/USDT:USDT"],
    "pair_blacklist": ["BNB/.*"]
  },

  "telegram": {
    "enabled": true,
    "token": "${TELEGRAM_TOKEN}",
    "chat_id": "${TELEGRAM_CHAT_ID}"
  },

  "api_server": {
    "enabled": true,
    "listen_ip_address": "0.0.0.0",
    "listen_port": 8080,
    "verbosity": "error",
    "enable_openapi": true,
    "jwt_secret_key": "${JWT_SECRET}",
    "username": "${FTRADE_USER}",
    "password": "${FTRADE_PASS}"
  },

  "bot_name": "freqtrade-dnodal0",
  "initial_state": "running",
  "force_entry_enable": false,
  "internals": {
    "process_throttle_secs": 5
  }
}
```

---

## Étape 5 : Interaction via MCP (bot en live)

Utiliser les outils MCP pour monitorer et interagir sans ligne de commande :

```
# Status du bot
mcp__freqtrade-mcp__fetch_bot_status

# Trades ouverts
mcp__freqtrade-mcp__fetch_trades

# Performance
mcp__freqtrade-mcp__fetch_performance
mcp__freqtrade-mcp__fetch_profit

# Balance
mcp__freqtrade-mcp__fetch_balance

# Forcer un trade (avec validation préalable)
mcp__freqtrade-mcp__place_trade

# Reload config sans restart
mcp__freqtrade-mcp__reload_config

# Gestion whitelist/blacklist
mcp__freqtrade-mcp__fetch_whitelist
mcp__freqtrade-mcp__add_blacklist
```

---

## Risk Management — Règles absolues

### Position sizing
```
risk_par_trade = capital_total × 0.01   # Max 1% du capital par trade
stake_amount = risk_par_trade / abs(stoploss)
```

### Règles opérationnelles
- **Max 3 trades simultanés** (max_open_trades: 3)
- **Stoploss max** : -3% par trade (jamais > -5%)
- **Daily loss limit** : arrêter le bot si -5% dans la journée
- **Ne jamais désactiver le stoploss** même si la stratégie semble "sure"
- **Futures uniquement** : utiliser la marge isolée, levier max 3x

### Levier recommandé par timeframe

| Timeframe | Levier max |
|-----------|-----------|
| 1m–5m     | 2x        |
| 15m–1h    | 3x        |
| 4h–1j     | 5x        |

---

## Docker Compose — Setup ARM64

```yaml
# docker-compose.yml
services:
  freqtrade:
    image: freqtradeorg/freqtrade:stable
    platform: linux/arm64
    restart: unless-stopped
    volumes:
      - ./user_data:/freqtrade/user_data
    ports:
      - "8080:8080"
    environment:
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_API_SECRET=${BINANCE_API_SECRET}
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade.log
      --db-url sqlite:////freqtrade/user_data/tradesv3.sqlite
      --config /freqtrade/user_data/config.json
      --strategy MyStrategy
```

---

## Debugging courant

| Problème | Cause probable | Solution |
|----------|---------------|---------|
| `No data found` | Données non téléchargées | `download-data` d'abord |
| `Insufficient funds` | Stake trop grand | Réduire `stake_amount` ou `tradable_balance_ratio` |
| `Pair not in whitelist` | Format incorrect pour futures | Utiliser `BTC/USDT:USDT` |
| `Strategy not found` | Fichier mal nommé | Classe Python = nom fichier = `--strategy` arg |
| ARM64 crash | Image incompatible | Ajouter `platform: linux/arm64` |
| Hyperopt lent | Manque de CPUs | `--jobs -1` utilise tous les cores |

---

## Checklist avant passage en live

- [ ] Backtest ≥ 1 an avec profit > 20% et drawdown < 15%
- [ ] Hyperopt validé sur données out-of-sample
- [ ] Dry-run ≥ 7 jours avec résultats cohérents avec backtest
- [ ] Config `.env` avec API keys (jamais dans config.json en clair)
- [ ] Telegram notifications configurées
- [ ] API server sécurisé (JWT + credentials)
- [ ] `dry_run: false` uniquement après checklist complète
- [ ] Stake initial ≤ 10% du capital total (phase de validation live)
