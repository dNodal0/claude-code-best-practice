---
name: ibkr-wheel-validator
description: >
  Validates Wheel Strategy options trades on IBKR (Cash-Secured Puts and Covered Calls).
  Use when user asks to "validate a wheel trade", "check a CSP", "check a covered call",
  "should I sell this put", "IV rank check", "premium % check", "early close options",
  "80% rule options", "wheel setup", or mentions DTE/delta/IV on equities/ETFs.
  Enforces IV Rank filter, DTE window, delta range, premium minimum, position sizing,
  and early-close 80-90% rule. Outputs GO / NO-GO with reasoning.
  Automatically fetches options data from IBKR Client Portal via Claude in Chrome
  when the user provides only a ticker — no manual data entry required.
metadata:
  author: dnodal0
  version: 1.1.0
  broker: IBKR
  strategy: Wheel (CSP → CC)
  instruments: US equities, ETFs (SPY, QQQ, individual stocks)
allowed-tools:
  - mcp__Claude_in_Chrome__navigate
  - mcp__Claude_in_Chrome__get_page_text
  - mcp__Claude_in_Chrome__read_page
  - mcp__Claude_in_Chrome__find
  - mcp__Claude_in_Chrome__javascript_tool
---

# IBKR Wheel Strategy Validator

## Context

Tu trades la Wheel Strategy sur IBKR : vente de Cash-Secured Puts (CSP) sur des actions/ETFs
que tu es prêt à détenir, puis vente de Covered Calls (CC) si assigné.
Objectif : générer du premium régulier tout en accumulant des positions solides.

---

## Étape 0 : Récupération automatique des données via Chrome

**Avant de demander quoi que ce soit à l'utilisateur**, tenter de récupérer les données
directement depuis le Client Portal IBKR ouvert dans Chrome.

### 0.1 — Vérifier si IBKR est ouvert

```
mcp__Claude_in_Chrome__navigate → https://clientportal.ibkr.com
mcp__Claude_in_Chrome__get_page_text
```

Si la page retourne une page de login → demander à l'utilisateur de se connecter d'abord,
puis relancer. Si déjà connecté → continuer.

### 0.2 — Naviguer vers la chaîne d'options du ticker

Une fois le ticker connu (demander UNIQUEMENT le ticker si rien n'est fourni) :

```
# Naviguer vers la page du ticker dans le portail
mcp__Claude_in_Chrome__navigate → https://clientportal.ibkr.com/portal/#/trade/options?symbol=TICKER

# Lire le contenu de la page
mcp__Claude_in_Chrome__read_page
```

Extraire automatiquement depuis la page :
- **Prix actuel** du sous-jacent
- **Chaîne d'options** : strikes disponibles autour du prix actuel
- **Premium bid/ask** pour chaque strike (calculer le mid)
- **Delta** par strike
- **DTE** par expiration disponible
- **IV implicite** (pour estimer IV Rank si disponible)

### 0.3 — Récupérer l'IV Rank depuis Market Chameleon

Si IV Rank non disponible directement sur IBKR :

```
mcp__Claude_in_Chrome__navigate → https://marketchameleon.com/Overview/TICKER/IV/
mcp__Claude_in_Chrome__get_page_text
```

Extraire : IV Rank 30 jours, IV actuelle, IV 52w high/low.

### 0.4 — Récupérer les positions ouvertes (Early-close check)

Pour vérifier si une position existante doit être fermée :

```
mcp__Claude_in_Chrome__navigate → https://clientportal.ibkr.com/portal/#/portfolio
mcp__Claude_in_Chrome__get_page_text
```

Extraire : positions options ouvertes, prix d'entrée, P&L actuel, DTE restant.

### 0.5 — Résumé de ce qui a été récupéré

Afficher un résumé des données collectées avant de lancer la validation :

```
📡 Données récupérées depuis IBKR + MarketChameleon :
  Ticker    : XXXX    Prix actuel : $XXX
  Strike    : $XXX    DTE : XX j   Premium mid : $X.XX
  Delta     : -0.XX   IV Rank : XX%
  Source    : IBKR Client Portal + MarketChameleon
```

Si certaines données sont manquantes → demander uniquement celles-là à l'utilisateur.

---

## Workflow de validation — OBLIGATOIRE avant toute exécution

### Étape 1 : Identifier le type de trade

Si les données Chrome sont complètes, passer directement à l'étape 2.
Sinon demander à l'utilisateur uniquement les données manquantes :
- Type : **CSP** ou **CC**
- Taille du compte allouée à ce sous-jacent

---

### Étape 2 : Filtre IV Rank

```
IV Rank = (IV actuelle - IV 52w low) / (IV 52w high - IV 52w low) × 100
```

| IV Rank | Signal |
|---------|--------|
| ≥ 30    | ✅ Bon contexte pour vendre du premium |
| 20–29   | ⚠️ Acceptable, premium réduit |
| < 20    | ❌ IV trop basse — attendre un spike |

**RÈGLE** : IV Rank < 20 → BLOQUER (premium insuffisant vs risque)

---

### Étape 3 : Filtre DTE (Days To Expiration)

| Type | DTE optimal | DTE limite |
|------|-------------|------------|
| CSP  | 30–45 jours | 21–60 jours |
| CC   | 21–35 jours | 14–45 jours |

**RÈGLES** :
- DTE < 14 → ❌ BLOQUER (gamma risk trop élevé)
- DTE > 60 → ❌ BLOQUER (capital immobilisé trop longtemps)
- DTE idéal : 30–45 jours (theta decay optimal)

---

### Étape 4 : Filtre Delta

| Type | Delta cible | Range acceptable |
|------|-------------|-----------------|
| CSP  | -0.20 à -0.30 | -0.15 à -0.35 |
| CC   | +0.20 à +0.30 | +0.15 à +0.35 |

**RÈGLES** :
- Delta > 0.35 (valeur absolue) → ❌ Trop risqué
- Delta < 0.15 → ❌ Premium insuffisant

---

### Étape 5 : Filtre Premium % (ROI mensuel)

```
premium_pct = premium_reçu / (strike × 100) × 100
roi_mensuel = premium_pct × (30 / DTE)
```

| ROI mensuel | Signal |
|-------------|--------|
| ≥ 1.5%      | ✅ Excellent |
| 1.0–1.4%    | ✅ Acceptable |
| 0.7–0.9%    | ⚠️ Faible mais OK en marché calme |
| < 0.7%      | ❌ BLOQUER |

**Objectif annuel** : 20–40% sur le capital alloué.

---

### Étape 6 : Strike vs Niveaux Techniques (CSP uniquement)

```
marge_sécurité = (prix_actuel - strike) / prix_actuel × 100
```

| Marge | Signal |
|-------|--------|
| ≥ 5%  | ✅ |
| 3–4%  | ⚠️ Acceptable si IV Rank élevé |
| < 3%  | ❌ Strike trop proche du prix |

---

### Étape 7 : Position Sizing

```
nb_contrats = floor(compte_total × 0.05 / (strike × 100))
```

**RÈGLES** :
- Max 5% du compte par sous-jacent
- Max 3 positions simultanées dans le même secteur
- En CC : 1 contrat par 100 actions détenues

---

### Étape 8 : Règle Early-Close 80-90%

```
profit_actuel = (premium_initial - premium_actuel) / premium_initial × 100
```

| Profit réalisé | Action |
|---------------|--------|
| ≥ 80%         | ✅ FERMER immédiatement |
| 50–79%        | 🔄 Surveiller |
| < 50%         | ⏳ Laisser courir si DTE > 14j |

**RÈGLE CRITIQUE** : Ne jamais attendre l'expiration pour les derniers 10-20% —
le gamma risk explose dans les 7 derniers jours.

---

## Output du validateur

```
╔══════════════════════════════════════════╗
║      IBKR WHEEL VALIDATOR v1.1           ║
╠══════════════════════════════════════════╣
║ Ticker       : XXXX                      ║
║ Type         : CSP / CC                  ║
║ Strike       : $XXX   Expiry: XX/XX      ║
║ DTE          : XX j   ✅/⚠️/❌          ║
║ IV Rank      : XX%    ✅/⚠️/❌          ║
║ Delta        : -0.XX  ✅/⚠️/❌          ║
║ Premium      : $X.XX  (X.XX% / mois)     ║
║ ROI mensuel  : X.XX%  ✅/⚠️/❌          ║
║ Marge strike : X.XX%  ✅/⚠️/❌          ║
║ Contrats     : X (capital: $XXXX)        ║
║ Source       : 🌐 IBKR Chrome / Manuel   ║
╠══════════════════════════════════════════╣
║ DÉCISION : ✅ TRADE VALIDÉ               ║
║         ou ❌ TRADE BLOQUÉ               ║
╚══════════════════════════════════════════╝
```

Si BLOQUÉ → expliquer quelle règle a échoué + suggestion corrective.
Si VALIDÉ → rappeler le target de close (80% du premium = $X.XX).

---

## Exemples d'invocation

### Mode automatique (Chrome ouvert sur IBKR)

```
/ibkr-wheel-validator NVDA CSP
```
→ Claude navigue sur IBKR, lit la chaîne d'options, récupère l'IV Rank sur
  MarketChameleon, et lance la validation sans autre input.

### Mode semi-automatique (Chrome dispo mais pas sur IBKR)

```
/ibkr-wheel-validator
```
→ Claude demande uniquement le ticker, puis va chercher le reste.

### Mode manuel (fallback si Chrome non disponible)

```
/ibkr-wheel-validator

CSP sur NVDA. Prix 875$, strike 820, 35 DTE, premium 4.20$,
delta -0.24, IV Rank 38, compte 40 000$
```

### Early-close check automatique

```
/ibkr-wheel-validator early-close
```
→ Claude lit le portfolio IBKR via Chrome, identifie toutes les positions
  options ouvertes, et signale celles qui ont atteint le seuil 80%.

---

## Checklist rapide

- [ ] IV Rank ≥ 20 (idéal ≥ 30)
- [ ] DTE entre 21 et 60 jours (idéal 30-45)
- [ ] Delta entre 0.15 et 0.35 (valeur absolue)
- [ ] ROI mensuel ≥ 0.7%
- [ ] Strike sous support technique (CSP)
- [ ] Position ≤ 5% du compte
- [ ] Pas d'earnings dans la fenêtre d'expiration
- [ ] Target early-close défini (80-90% du premium)

---

## Gestion post-assignation (Wheel complet)

**Si assigné sur CSP** → tu détiens 100 actions au prix du strike
1. Prix de revient effectif = strike - premium_reçu
2. Vendre une CC au strike = prix de revient (ou légèrement au-dessus)
3. Objectif CC : récupérer 1-2% supplémentaire

**Si appelé sur CC** → actions vendues au strike
1. Calculer ROI total (premium CSP + premium CC + éventuelle PV)
2. Recommencer le cycle
