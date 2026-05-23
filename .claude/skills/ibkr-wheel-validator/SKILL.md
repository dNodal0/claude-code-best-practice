---
name: ibkr-wheel-validator
description: >
  Validates Wheel Strategy options trades on IBKR (Cash-Secured Puts and Covered Calls).
  Use when user asks to "validate a wheel trade", "check a CSP", "check a covered call",
  "should I sell this put", "IV rank check", "premium % check", "early close options",
  "80% rule options", "wheel setup", or mentions DTE/delta/IV on equities/ETFs.
  Enforces IV Rank filter, DTE window, delta range, premium minimum, position sizing,
  and early-close 80-90% rule. Outputs GO / NO-GO with reasoning.
metadata:
  author: dnodal0
  version: 1.0.0
  broker: IBKR
  strategy: Wheel (CSP → CC)
  instruments: US equities, ETFs (SPY, QQQ, individual stocks)
---

# IBKR Wheel Strategy Validator

## Context

Tu trades la Wheel Strategy sur IBKR : vente de Cash-Secured Puts (CSP) sur des actions/ETFs
que tu es prêt à détenir, puis vente de Covered Calls (CC) si assigné.
Objectif : générer du premium régulier tout en accumulant des positions solides.

---

## Workflow de validation — OBLIGATOIRE avant toute exécution

### Étape 1 : Identifier le type de trade

Demander à l'utilisateur :
- Type : **CSP** (Cash-Secured Put) ou **CC** (Covered Call)
- Ticker + prix actuel du sous-jacent
- Strike envisagé
- Expiration (date ou DTE)
- Premium reçu (bid/mid)
- IV Rank actuel (si connu)
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

Si IV Rank non fourni : demander ou estimer via contexte (VIX, actualités récentes).

---

### Étape 3 : Filtre DTE (Days To Expiration)

| Type | DTE optimal | DTE limite |
|------|-------------|------------|
| CSP  | 30–45 jours | 21–60 jours |
| CC   | 21–35 jours | 14–45 jours |

**RÈGLES** :
- DTE < 14 → ❌ BLOQUER (gamma risk trop élevé, peu de valeur temps restante)
- DTE > 60 → ❌ BLOQUER (trop d'exposition temporelle, capital immobilisé)
- DTE idéal : 30–45 jours (theta decay optimal, zone 45 DTE de Tastytrade)

---

### Étape 4 : Filtre Delta

| Type | Delta cible | Range acceptable |
|------|-------------|-----------------|
| CSP  | -0.20 à -0.30 | -0.15 à -0.35 |
| CC   | +0.20 à +0.30 | +0.15 à +0.35 |

**RÈGLES** :
- Delta > 0.35 (en valeur absolue) → ❌ Trop risqué, probabilité d'assignation trop haute
- Delta < 0.15 → ❌ Premium trop faible, ROI insuffisant

---

### Étape 5 : Filtre Premium % (ROI mensuel)

```
premium_pct = premium_reçu / (strike × 100) × 100
roi_mensuel = premium_pct × (30 / DTE)
```

| ROI mensuel annualisé | Signal |
|----------------------|--------|
| ≥ 1.5% / mois        | ✅ Excellent |
| 1.0–1.4% / mois      | ✅ Acceptable |
| 0.7–0.9% / mois      | ⚠️ Faible mais OK en marché calme |
| < 0.7% / mois        | ❌ BLOQUER — ne compense pas le risque |

**Objectif annuel** : 20–40% sur le capital alloué via accumulation de premium.

---

### Étape 6 : Strike vs Niveaux Techniques (CSP uniquement)

Pour les CSPs, vérifier que le strike est sous un support technique solide :

```
marge_sécurité = (prix_actuel - strike) / prix_actuel × 100
```

| Marge | Signal |
|-------|--------|
| ≥ 5%  | ✅ Strike bien en dessous du prix actuel |
| 3–4%  | ⚠️ Acceptable si IV Rank élevé |
| < 3%  | ❌ Strike trop proche du prix — risque d'assignation élevé |

Demander : y a-t-il un support technique identifié sous le strike ?

---

### Étape 7 : Position Sizing

```
capital_max_par_trade = min(
  compte_total × 0.05,        # Max 5% du compte par position
  strike × 100 × nb_contrats  # Exposition réelle
)
nb_contrats = floor(capital_alloué / (strike × 100))
```

**RÈGLES** :
- Max 5% du compte par sous-jacent en CSP
- Max 3 positions simultanées en Wheel sur le même secteur (corrélation)
- En CC après assignation : 1 contrat par 100 actions détenues

---

### Étape 8 : Règle Early-Close 80-90%

```
profit_actuel = (premium_initial - premium_actuel) / premium_initial × 100
```

| Profit réalisé | Action recommandée |
|---------------|-------------------|
| ≥ 80%         | ✅ FERMER la position — objectif atteint |
| 50–79%        | 🔄 Surveiller, approcher du target |
| < 50%         | ⏳ Laisser courir si DTE > 14 jours |

**RÈGLE CRITIQUE** : Ne jamais laisser une option expirer pour capturer les derniers 10-20%
de premium — le gamma risk explose dans les 7 derniers jours.

---

## Output du validateur

Toujours afficher ce résumé :

```
╔══════════════════════════════════════════╗
║      IBKR WHEEL VALIDATOR v1.0           ║
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
╠══════════════════════════════════════════╣
║ DÉCISION : ✅ TRADE VALIDÉ               ║
║         ou ❌ TRADE BLOQUÉ               ║
╚══════════════════════════════════════════╝
```

Si BLOQUÉ → expliquer quelle règle a échoué + suggestion corrective.
Si VALIDÉ → rappeler le target de close (80% du premium = $X.XX).

---

## Exemples

### Exemple 1 : CSP valide sur SPY

User : "SPY à 520$, vendre CSP strike 500, expiry 35 DTE, premium 2.80, IV Rank 42, compte 50k$"

Calculs :
1. IV Rank 42% ✅
2. DTE 35j ✅
3. Delta estimé ~-0.22 ✅
4. Premium% = 2.80/(500×100)×100 = 0.56% → ROI mensuel = 0.56%×(30/35) = 0.48%/mois ⚠️ faible
5. Marge = (520-500)/520 = 3.8% ⚠️
6. Contrats = floor(50000×0.05/(500×100)) = floor(2500/50000) = 0 → 1 contrat max

Note : ROI faible en marché calme. Acceptable si SPY = conviction hold.

Résultat : TRADE VALIDÉ (avec réserve sur ROI) — 1 contrat, close à $0.56 (80%)

### Exemple 2 : CSP bloqué — IV trop basse

User : "AAPL à 185$, CSP strike 180, 30 DTE, premium 0.90, IV Rank 15"

IV Rank 15% ❌ → BLOQUÉ. Attendre spike IV (earnings, correction marché).

### Exemple 3 : Early close

User : "J'ai vendu un CSP TSLA pour 3.50$ il y a 20 jours, il vaut maintenant 0.60$"

Profit = (3.50-0.60)/3.50 = 82.9% ✅ → FERMER immédiatement. Target 80% atteint.
Capital libéré pour un nouveau trade.

---

## Checklist rapide avant exécution

- [ ] IV Rank ≥ 20 (idéal ≥ 30)
- [ ] DTE entre 21 et 60 jours (idéal 30-45)
- [ ] Delta entre 0.15 et 0.35 (valeur absolue)
- [ ] ROI mensuel ≥ 0.7%
- [ ] Strike sous support technique (CSP) ou au-dessus résistance (CC)
- [ ] Position ≤ 5% du compte
- [ ] Pas d'earnings dans la fenêtre d'expiration (vérifier IBKR calendar)
- [ ] Target early-close défini (80-90% du premium)

---

## Gestion post-assignation (Wheel complet)

**Si assigné sur CSP** → tu détiens 100 actions au prix du strike
1. Prix de revient effectif = strike - premium_reçu
2. Vendre immédiatement une CC au strike = prix de revient (ou légèrement au-dessus)
3. Objectif CC : récupérer 1-2% supplémentaire
4. Si actions remontent → close CC ou laisser expirer OTM

**Si appelé sur CC** → actions vendues au strike
1. Cycle terminé : calculer ROI total (premium CSP + premium CC + éventuelle plus-value)
2. Recommencer le cycle sur ce ticker ou passer à un autre
