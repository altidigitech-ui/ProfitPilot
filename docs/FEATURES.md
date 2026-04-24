# FEATURES.md — Spécification des 41 features PilotProfit

> **Référence unique pour chaque feature.**
> **Avant d'implémenter une feature, vérifie sa spec ici.**

---

## LÉGENDE

- **Plan** : Free / Starter / Pro / Enterprise — plan minimum requis
- **Agent** : agent spécialiste responsable (Profit Analyst / Fraud Investigator / Chargeback Specialist / Data Integrity / Ads Sync)
- **Analyzer** : fichier analyzer dans `backend/app/agent/analyzers/`
- **Endpoint** : route API qui expose les résultats
- **Composant** : composant frontend principal
- **Phase** : M1 (launch) = prioritaire / M2+ = après launch
- **Cross-LLM Review** : indique si la feature déclenche une review high-stakes

---

## PLAN CHECKING

Avant d'exécuter une feature, TOUJOURS vérifier le plan du merchant :

```python
from app.services.billing import check_plan_access

async def run_analyzer(merchant_id: str, feature: str):
    if not await check_plan_access(merchant_id, feature):
        raise AppError(
            code=ErrorCode.PLAN_REQUIRED,
            message=f"Feature '{feature}' requires {FEATURE_PLANS[feature]} plan or above",
            status_code=403,
            context={"feature": feature, "required_plan": FEATURE_PLANS[feature]},
        )
```

Mapping complet : `docs/PRICING_PLANS.md`.

---

## ORDRE D'IMPLEMENTATION RECOMMANDÉ (M1 launch)

1. **Data Integrity Check** (#10) — fondation : sans données fiables, rien ne marche
2. **True Profit** (#1) + **Daily P&L** (#3) — le noyau du module Profit
3. **Ad Spend Tracker 100%** (#5) — car True Profit sans ad spend = profit mensonger
4. **Pre-Ship Score** (#16) + **Smart Hold** (#17) — le noyau du module Anti-Fraude
5. **Auto-Evidence Builder** (#18) + **Approval Workflow** — différenciation #1 vs Chargeflow
6. **Ratio Monitor** (#22) — critique pour éviter le ban Visa/MC
7. Le reste suit par ordre de priorité des plans (Starter → Pro → Enterprise)

---

# MODULE 1 : PROFIT (15 features)

---

### #1 — True Profit

| Champ | Valeur |
|-------|--------|
| Plan | Free (basique) / Starter (complet) |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `true_profit.py` |
| Endpoint | `GET /api/v1/profit/orders/{order_id}` / `GET /api/v1/profit/products` |
| Composant | `dashboard/profit/TrueProfitTable.tsx` |
| Cross-LLM Review | Non (calcul déterministe) |

**Input :** Order Shopify (revenue, items, taxes, shipping facturé), COGS depuis Shopify `inventory_items.cost` ou override merchant, ad spend attribué (via `ad_spend.py`), transaction fees Stripe, apps cost allocated (`app_cost.py`), landed cost (`landed_cost.py`), retour cost si applicable.

**Output :**
```json
{
  "order_id": "gid://shopify/Order/5234567890",
  "currency": "USD",
  "revenue_cents": 18500,
  "costs": {
    "cogs_cents": 4200,
    "shipping_cost_cents": 890,
    "payment_fees_cents": 569,
    "ad_spend_allocated_cents": 2300,
    "apps_cost_allocated_cents": 87,
    "duties_cents": 144,
    "refund_cost_cents": 0
  },
  "total_costs_cents": 8190,
  "gross_profit_cents": 14300,
  "gross_margin_pct": 77.3,
  "net_profit_cents": 10310,
  "net_margin_pct": 55.7,
  "calculated_at": "2026-04-24T01:00:00Z",
  "data_integrity": { "verified": true, "variance_pct": 0.3 }
}
```

**Logique :**
- `gross_profit = revenue - cogs - shipping_cost - payment_fees`
- `net_profit = gross_profit - ad_spend_allocated - apps_cost_allocated - duties - refund_cost`
- Ad spend attribué via règle merchant-configurable : même canal (Meta→Meta) ou cross-canal pondéré par ROAS historique.
- Apps cost : abonnement mensuel Shopify divisé par nb orders/mois (moyenne 30j).
- Formules détaillées dans `docs/PROFIT_FORMULAS.md`.

**Edge cases :**
- COGS manquant → feature en mode dégradé : gross seul, flag `missing_cogs: true`, UI demande au merchant de saisir
- Multi-currency order → conversion via `services/fx_rates.py` au taux du jour de l'order (pas du jour actuel)
- Order partiellement refund → recalcul `refund_cost_cents` avec duty et shipping déjà payés
- Order avec post-purchase upsell → upsell traité comme line item séparé avec son propre profit calc
- Bot order détecté (via Fraud Investigator) → exclu du P&L, flag `excluded: "bot"`

---

### #2 — App Cost Allocator

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `app_cost.py` |
| Endpoint | `GET /api/v1/profit/apps/cost-breakdown` |
| Composant | `dashboard/profit/AppCostBreakdown.tsx` |
| Cross-LLM Review | Non |

**Input :** Shopify app charges (recurring_application_charges API), orders count des 30 derniers jours.

**Output :**
```json
{
  "period": "last_30d",
  "total_apps_cost_cents": 28700,
  "orders_count": 412,
  "avg_cost_per_order_cents": 69,
  "apps": [
    {
      "app_handle": "klaviyo",
      "name": "Klaviyo",
      "monthly_cost_cents": 9500,
      "cost_per_order_cents": 23,
      "worth_it_score": 0.78,
      "revenue_attributed_cents": 145000
    }
  ]
}
```

**Logique :** Pour chaque app avec recurring charge : `cost_per_order = monthly_cost / orders_count_last_30d`. Score "worth it" = revenue attribué à l'app (si trackable via UTM ou tag) / cost. Permet au merchant de voir quelles apps sont rentables.

**Edge cases :**
- App avec usage-based charge → inclure les `usage_charges` en plus du recurring
- App désinstallée mais encore facturée ("ghost billing") → alerte séparée via #11 Proactive Bug Alert
- App gratuite → `cost_per_order: 0`, toujours incluse dans la liste pour transparence

---

### #3 — Daily P&L

| Champ | Valeur |
|-------|--------|
| Plan | Free (basique — 7 derniers jours) / Starter (illimité) |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `true_profit.py` (agrégé) |
| Endpoint | `GET /api/v1/profit/pnl?from=2026-04-01&to=2026-04-24` |
| Composant | `dashboard/profit/PnLChart.tsx` |
| Cross-LLM Review | Non |

**Input :** Tous les orders de la période, ad spend daily aggregated, OPEX merchant (catégorisé via #7).

**Output :**
```json
{
  "period": { "from": "2026-04-01", "to": "2026-04-24" },
  "currency": "USD",
  "days": [
    {
      "date": "2026-04-24",
      "revenue_cents": 245000,
      "cogs_cents": 58000,
      "shipping_cents": 12000,
      "payment_fees_cents": 7500,
      "ad_spend_cents": 45000,
      "apps_cost_cents": 950,
      "duties_cents": 1800,
      "opex_cents": 8200,
      "gross_profit_cents": 167500,
      "net_profit_cents": 111550,
      "orders_count": 38,
      "refunds_count": 2
    }
  ],
  "totals": { ... }
}
```

**Logique :** Nightly cron Celery (`tasks/profit_tasks.py::nightly_pnl`) agrège tous les orders du jour, pull ad spend daily de chaque canal (Meta/Google/TikTok), calcul tombé en DB `daily_pnl`. Endpoint retourne les jours déjà calculés (pas de calcul on-the-fly au-delà du jour J).

**Edge cases :**
- Jour J en cours → flag `partial: true`, recalculé toutes les heures
- Intégration Meta down → ad spend marqué `estimated` avec baseline des 7 derniers jours, alerte via Ads Sync Agent
- Orders en pending_payment → exclus jusqu'à capture

---

### #4 — Margin Alerts

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `margin_monitor.py` |
| Endpoint | `GET /api/v1/profit/margins/alerts` / `POST /api/v1/profit/margins/thresholds` |
| Composant | `dashboard/profit/MarginAlerts.tsx` |
| Cross-LLM Review | Oui, pour margin breach > 50% de drop vs baseline |

**Input :** Margin par produit (7 derniers jours vs baseline 30j), seuils configurables par le merchant (default : alerte si margin < 20% ou drop > 30% vs baseline).

**Output :**
```json
{
  "alerts": [
    {
      "product_id": "gid://shopify/Product/12345",
      "sku": "TSHIRT-BLK-L",
      "current_margin_pct": 18.5,
      "baseline_margin_pct": 42.1,
      "drop_pct": 56.1,
      "likely_cause": "cogs_increase",
      "detected_at": "2026-04-24T08:00:00Z",
      "cross_llm_reviewed": true,
      "reviewer_consensus": true
    }
  ]
}
```

**Logique :** Compare margin actuelle vs baseline 30j du produit. Claude Opus génère `likely_cause` ("cogs_increase", "ad_spend_spike", "shipping_cost_rise", "promo_discount_unexpected"). Si drop > 50%, trigger cross-LLM review pour valider la cause avant d'alerter le merchant (évite les faux positifs qui détruisent la confiance).

**Edge cases :**
- Produit avec < 10 orders sur 30j → pas de baseline fiable, alerte différée
- Produit saisonnier → Mem0 apprend la saisonnalité, baseline saisonnière utilisée
- Promo active → alerte silencée pendant la durée de la promo (configurable)

---

### #5 — Ad Spend Tracker (100%)

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Ads Sync |
| Analyzer | `ad_spend.py` |
| Endpoint | `GET /api/v1/ads/spend` / `GET /api/v1/ads/spend/by-channel` |
| Composant | `dashboard/ads/AdSpendChart.tsx` |
| Cross-LLM Review | Non |

**Input :** Meta Marketing API (`act_{id}/insights` avec `level=account`), Google Ads API (`CustomerService.GetAccountSummary`), TikTok Business API (`/report/integrated/get/`), toutes les data journalières.

**Output :**
```json
{
  "period": "last_30d",
  "total_spend_cents": 1250000,
  "by_channel": {
    "meta": { "spend_cents": 620000, "campaigns_count": 12, "last_sync": "2026-04-24T06:00:00Z" },
    "google": { "spend_cents": 480000, "campaigns_count": 8, "last_sync": "2026-04-24T06:00:00Z" },
    "tiktok": { "spend_cents": 150000, "campaigns_count": 3, "last_sync": "2026-04-24T06:00:00Z" }
  },
  "discrepancy_alerts": [
    {
      "channel": "google",
      "reported_in_shopify_cents": 72000,
      "actual_from_api_cents": 480000,
      "variance_pct": 84.4,
      "severity": "critical"
    }
  ]
}
```

**Logique :** Pull daily des 3 APIs via cron Celery (`tasks/ads_tasks.py::daily_ad_spend_sync`). Comparaison avec ce que Shopify UTM tracking rapporte → **alerte si écart > 10%** (faille BeProfit exploitée : ils ne trackent que 15% du Google Ads spend).

**Edge cases :**
- Token OAuth expiré → marque l'intégration `unhealthy`, alerte via Integration Health (#integration-health), ne remplace PAS par une estimation (on ne ment pas)
- Meta Conversions API vs UTM → on privilégie l'API direct, on loggue les deux pour debug
- Compte Ads à 0 spend depuis 7j → pas d'alerte de "data manquante", on considère que le merchant a pausé

---

### #6 — Return Cost Calculator

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `return_cost.py` |
| Endpoint | `GET /api/v1/profit/returns/cost-breakdown` |
| Composant | `dashboard/profit/ReturnCost.tsx` |
| Cross-LLM Review | Non |

**Input :** Shopify refunds API, original order, shipping cost retour (si merchant a un carrier intégré), restocking fees merchant, stock disposition (resellable / damaged / dead).

**Output :**
```json
{
  "return_id": "gid://shopify/Refund/98765",
  "order_id": "gid://shopify/Order/5234567890",
  "refund_amount_cents": 8500,
  "real_cost": {
    "product_value_lost_cents": 4200,
    "return_shipping_cents": 890,
    "original_shipping_cents_unrecoverable": 890,
    "payment_fee_unrecoverable_cents": 569,
    "duty_unrecoverable_cents": 144,
    "restocking_labor_estimate_cents": 200,
    "dead_stock_cents": 0
  },
  "total_real_cost_cents": 6893,
  "note": "Real cost 81% of refund amount — return is more expensive than refund suggests"
}
```

**Logique :** Le merchant voit souvent "retour de 85$" et pense perdre 85$. La réalité : il a déjà payé les Stripe fees, les duties d'import (non-refondables), parfois le shipping aller, plus maintenant le shipping retour + labor + stock possiblement invendable. Formule dans `PROFIT_FORMULAS.md`.

**Edge cases :**
- Retour partiel (1 item sur 3) → allocation proportionnelle des coûts
- Produit non-récupéré (lost in transit return) → 100% du COGS compté comme perte
- Return avec discount appliqué à l'origine → recalcul de la part discountée

---

### #7 — OPEX Categorizer

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `opex_categorizer.py` |
| Endpoint | `GET /api/v1/profit/opex` / `POST /api/v1/profit/opex/manual` |
| Composant | `dashboard/profit/OpexCategorizer.tsx` |
| Cross-LLM Review | Non |

**Input :** Shopify app charges, Stripe fees détaillés, QuickBooks/Xero si connecté, entrées manuelles merchant (abonnements tiers, salaires, loyer, outils).

**Output :**
```json
{
  "period": "2026-04",
  "categories": {
    "software": { "total_cents": 42300, "items_count": 12, "auto_categorized_pct": 100 },
    "payment_processing": { "total_cents": 28500, "items_count": 412 },
    "advertising": { "total_cents": 1250000, "items_count": 23 },
    "shipping_operations": { "total_cents": 95000, "items_count": 412 },
    "team": { "total_cents": 450000, "items_count": 2, "auto_categorized_pct": 0 },
    "uncategorized": { "total_cents": 3200, "items_count": 2 }
  }
}
```

**Logique :** Auto-categorization via Claude Opus (rules + context Mem0). Catégories IRS Schedule C compatibles (prep direct pour tax export #8). Uncategorized demande action du merchant.

**Edge cases :**
- Charge ambiguë (ex: "Stripe Atlas — $500") → Claude propose 2 catégories, merchant choisit
- Charge récurrente déjà catégorisée manuellement → Mem0 retient pour les mois suivants
- Devise étrangère → conversion au taux du jour de la charge

---

### #8 — Tax Ready Export

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | N/A (actor `tax_export.py`) |
| Endpoint | `GET /api/v1/tax/export/{format}` (format=quickbooks/xero/csv) |
| Composant | `dashboard/reports/TaxExport.tsx` |
| Cross-LLM Review | Non |

**Input :** Toute la data de l'année (revenue, costs, OPEX, returns, chargebacks, ad spend, duties), catégorisées et pré-formatées.

**Output :** Fichier téléchargeable
- `quickbooks` : IIF ou CSV import-ready avec les comptes pré-mappés
- `xero` : CSV format Xero avec tracking categories
- `csv` : générique pour comptable (colonnes Date, Category, Description, Amount, Currency, Tax)

**Logique :** Génération async via Celery (`report_tasks::tax_export`). Le merchant reçoit un email quand prêt (délai ~2 min pour 1 an de data). Export daté, signé (checksum), versionné (si re-export, v2 distinct).

**Edge cases :**
- Données incomplètes (ex: COGS manquants sur 12% des orders) → export bloqué avec rapport "à corriger avant export"
- Multi-currency → chaque transaction dans sa currency d'origine + colonne USD converted
- Stores multi-entités (LLC différentes) → export séparé par entité

---

### #9 — Cashflow Forecast 30/60/90j

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `cashflow.py` |
| Endpoint | `GET /api/v1/profit/cashflow-forecast?horizon=30d` |
| Composant | `dashboard/profit/CashflowForecast.tsx` |
| Cross-LLM Review | Oui, pour horizon 90j (more uncertainty) |

**Input :** Revenue des 90j passés (seasonality via Mem0), recurring costs (apps, team), upcoming payables (Stripe payout schedule, supplier POs si QuickBooks/Xero connecté), saisonnalité historique.

**Output :**
```json
{
  "horizon_days": 30,
  "starting_balance_cents": 4500000,
  "projected": [
    {
      "date": "2026-04-25",
      "expected_revenue_cents": 285000,
      "expected_costs_cents": 198000,
      "projected_balance_cents": 4587000,
      "confidence": 0.82
    }
  ],
  "risk_flags": [
    {
      "type": "low_cash_warning",
      "date": "2026-05-12",
      "projected_balance_cents": 380000,
      "message": "Projected balance low — consider delaying non-urgent spend or negotiating supplier terms"
    }
  ]
}
```

**Logique :** Régression simple (rolling average 30d + seasonal multipliers) pour revenue. Costs : somme des recurring + projected variable (ad spend à scaling constant, COGS à %-revenue constant). Confidence décroît avec l'horizon (0.9 à J+7, 0.6 à J+90).

**Edge cases :**
- Store < 90 jours d'historique → forecast limité à 7j avec warning "Not enough history"
- Event exceptionnel détecté (ex: BFCM) → modèle alerte "Baseline skewed, manual override suggested"
- Ad spend très volatile (±50% d'une semaine à l'autre) → confidence divisée par 2

---

### #10 — Data Integrity Check

| Champ | Valeur |
|-------|--------|
| Plan | Free |
| Phase | M1 (FONDATION — à implémenter en premier) |
| Agent | Data Integrity |
| Analyzer | `data_integrity.py` |
| Endpoint | `GET /api/v1/integrity/checks` / `GET /api/v1/integrity/alerts` |
| Composant | `dashboard/shared/DataIntegrityBadge.tsx` |
| Cross-LLM Review | Non (règles déterministes) |

**Input :** Shopify Orders API, Stripe Balance Transactions, Meta/Google Ads insights, notre DB.

**Output :**
```json
{
  "last_check_at": "2026-04-24T01:00:00Z",
  "checks": [
    {
      "source_a": "shopify_orders_count_30d",
      "source_b": "our_db_orders_count_30d",
      "a_value": 412,
      "b_value": 412,
      "variance_pct": 0.0,
      "status": "ok"
    },
    {
      "source_a": "shopify_revenue_30d",
      "source_b": "stripe_charges_30d",
      "a_value_cents": 12450000,
      "b_value_cents": 12698000,
      "variance_pct": 1.99,
      "status": "ok"
    },
    {
      "source_a": "meta_ads_spend_30d_api",
      "source_b": "meta_ads_spend_30d_shopify_utm",
      "a_value_cents": 620000,
      "b_value_cents": 93000,
      "variance_pct": 85.0,
      "status": "critical"
    }
  ]
}
```

**Logique :** Hourly cron (`tasks/integrity_tasks::hourly_reconciliation`). Compare cross-sources. **Règle absolue :** si variance > 2% sur une paire critique → `AppError(IntegrityError)` et on alerte le merchant plutôt que d'afficher une donnée qu'on sait fausse. C'est l'inverse de TrueProfit (7 reviews 1★ pour "données fausses non communiquées").

**Edge cases :**
- Source down (ex: Meta API 5xx) → statut `pending`, pas `critical`
- Fenêtre de timing (orders créés dans les dernières 5 minutes) → exclus pour éviter les faux positifs de latence
- Refunds en transit → Shopify les affiche immédiatement, Stripe avec delay 24-48h → tolérance adaptée

**Détail complet :** `docs/DATA_INTEGRITY.md`

---

### #11 — Proactive Bug Alert

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Data Integrity |
| Analyzer | `data_integrity.py` (sous-routine) |
| Endpoint | `GET /api/v1/integrity/alerts?proactive=true` |
| Composant | `dashboard/shared/ProactiveAlertBanner.tsx` |
| Cross-LLM Review | Non |

**Input :** Anomalies détectées par Data Integrity Check, logs d'erreurs de nos analyzers.

**Output :** Notification push + email + banner in-app.
```json
{
  "alert_id": "alert_abc123",
  "type": "ghost_billing_detected",
  "severity": "high",
  "message": "L'app 'OldReviews' est désinstallée depuis 14 jours mais continue de vous facturer 19$/mois. Contactez Shopify Billing pour un refund.",
  "action_url": "https://admin.shopify.com/admin/billing",
  "detected_at": "2026-04-24T09:15:00Z"
}
```

**Logique :** Détecte **AVANT que le merchant le voie** : apps désinstallées qui facturent encore, COGS soudainement à 0 (import CSV cassé), un canal d'ads qui ne sync plus depuis 48h, etc. Exploite la faille TrueProfit : "they don't communicate bugs".

**Edge cases :**
- Alerte déjà envoyée il y a < 24h → dedup, pas de doublon
- Merchant en pause (vacation mode Shopify) → alertes non-critiques différées

---

### #12 — LTV propre (sans bots / non-acheteurs)

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `ltv_clean.py` |
| Endpoint | `GET /api/v1/profit/ltv` |
| Composant | `dashboard/profit/LTVCurve.tsx` |
| Cross-LLM Review | Non |

**Input :** Customer base Shopify, orders history, fraud detection flags (pour exclure bots identifiés par Fraud Investigator), email engagement data (si Klaviyo connecté).

**Output :**
```json
{
  "cohort": "2025-Q4",
  "customers_total": 1240,
  "customers_real_buyers": 987,
  "bots_excluded": 142,
  "non_buyers_excluded": 111,
  "ltv_cents": {
    "30d": 4850,
    "90d": 7200,
    "180d": 9800,
    "365d": 13400
  },
  "repeat_purchase_rate": 0.34,
  "avg_order_value_cents": 6200
}
```

**Logique :** LTV = revenue moyen par customer réel, sur une fenêtre temporelle donnée. Exclut : customers marqués bot par Fraud Investigator (#16), customers sans achat (faille TrueProfit : "LTV includes non-buyers"). Formule détaillée dans `PROFIT_FORMULAS.md`.

**Edge cases :**
- Customer unique qui fait 50 orders (B2B wholesale) → flag, peut être inclus ou exclu selon config merchant
- Refund total sur une commande → order exclu du LTV de ce customer
- Customer reactivé après 18 mois → repart dans une nouvelle cohort

---

### #13 — Multi-Fulfillment Sync

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `true_profit.py` (enrichi multi-fulfillment) |
| Endpoint | `GET /api/v1/profit/fulfillment-breakdown` |
| Composant | `dashboard/profit/FulfillmentBreakdown.tsx` |
| Cross-LLM Review | Non |

**Input :** Shopify Fulfillments API (locations multiples), ShipStation API, ShipBob API, Amazon MCF API, manual warehouse data.

**Output :**
```json
{
  "period": "last_30d",
  "fulfillment_sources": [
    {
      "source": "shipbob",
      "location_id": "shipbob_wh_1",
      "orders_fulfilled": 285,
      "shipping_cost_cents": 67200,
      "avg_cost_per_order_cents": 236,
      "storage_fees_cents": 12000,
      "total_cost_cents": 79200
    },
    {
      "source": "amazon_mcf",
      "orders_fulfilled": 98,
      "shipping_cost_cents": 42100,
      "avg_cost_per_order_cents": 429
    },
    {
      "source": "shopify_self",
      "orders_fulfilled": 29,
      "shipping_cost_cents": 4800
    }
  ],
  "discrepancy_flags": []
}
```

**Logique :** Tire les coûts réels de chaque source (vs ce que Shopify affiche qui est parfois juste le shipping facturé au client). Calcul du coût total par fulfillment source incluant storage, pick & pack, receiving. Exploite la faille Lifetimely/AMP : "COGS incorrect avec fulfillment multi-source".

**Edge cases :**
- Order split entre 2 warehouses → coût alloué au prorata des items
- Sources sans API publique (3PL custom) → CSV upload mensuel + allocation proportionnelle
- Retour vers un warehouse différent de l'origine → coût retour tracké séparément

---

### #14 — Post-Purchase Upsell Tracker

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `true_profit.py` (logique dédiée) |
| Endpoint | `GET /api/v1/profit/upsells` |
| Composant | `dashboard/profit/UpsellPerformance.tsx` |
| Cross-LLM Review | Non |

**Input :** Orders Shopify avec `source_identifier` indiquant post-purchase, apps d'upsell (ReConvert, AfterSell, Zipify) via leurs webhooks ou tags d'order.

**Output :**
```json
{
  "period": "last_30d",
  "upsells": {
    "total_upsell_orders": 87,
    "total_upsell_revenue_cents": 145000,
    "total_upsell_net_profit_cents": 89000,
    "avg_upsell_margin_pct": 61.4,
    "acceptance_rate_pct": 21.1
  },
  "by_product": [
    {
      "sku": "ADDON-WARRANTY",
      "offered_count": 412,
      "accepted_count": 87,
      "revenue_cents": 87000,
      "net_profit_cents": 76000,
      "net_margin_pct": 87.4
    }
  ]
}
```

**Logique :** Les post-purchase upsells sont souvent mal trackés par les profit trackers classiques (faille AMP/Lifetimely : "post-purchase upsells cassent les chiffres"). On identifie chaque upsell comme un revenue séparé, avec son propre COGS et profit, attribué à l'order original.

**Edge cases :**
- Upsell refused → comptage dans `offered_count` mais pas dans `accepted_count`
- Upsell refund → profit recalculé
- App d'upsell non-connectée → détection via tags d'order ou `note_attributes`

---

### #15 — Currency FX Loss Tracker (exclusivité)

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `fx_loss.py` |
| Endpoint | `GET /api/v1/profit/fx-loss` |
| Composant | `dashboard/profit/FXLossCard.tsx` |
| Cross-LLM Review | Non |

**Input :** Orders multi-currency (Shopify `presentment_currency` vs `shop_currency`), Stripe settlement currency vs order currency, taux FX du jour de l'order (`services/fx_rates.py`), taux FX du jour du settlement.

**Output :**
```json
{
  "period": "last_30d",
  "shop_currency": "USD",
  "presentment_currencies": ["EUR", "GBP", "CAD"],
  "total_orders_multi_currency": 142,
  "total_loss_cents_usd": 28500,
  "by_currency": [
    {
      "currency": "EUR",
      "orders_count": 78,
      "gross_order_value_local": 45000,
      "gross_order_value_usd_at_order_time": 48600,
      "settled_value_usd": 47800,
      "fx_loss_cents_usd": 80000,
      "avg_loss_pct": 1.65
    }
  ],
  "recommendation": "Consider multi-currency payout via Shopify Payments or Wise Business to reduce FX spread (~2-3% savings)"
}
```

**Logique :** Aucun concurrent Shopify ne track ça. Entre le moment où un customer paie en EUR et le moment où Stripe settle en USD, il y a un spread de 1-3% que personne ne voit. `fx_loss = order_value_usd_at_order_time - settled_value_usd`. Claude Opus génère la recommendation contextuelle.

**Edge cases :**
- Shopify Payments avec settlement multi-currency → loss différent (Shopify applique son propre spread)
- Refund dans une currency ≠ original → double spread possible, tracké séparément
- Currency volatile (ex: ARS, TRY) → Mem0 flag "high volatility" et la recommendation change

---

# MODULE 2 : ANTI-FRAUDE (18 features)

---

### #16 — Pre-Ship Score

| Champ | Valeur |
|-------|--------|
| Plan | Free (basique 0-100 sans raisons) / Starter (complet avec reasons) |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `fraud_scorer.py` |
| Endpoint | `GET /api/v1/fraud/score/{order_id}` |
| Composant | `dashboard/fraud/RiskScore.tsx` |
| Cross-LLM Review | Non (le scoring est déterministe, la décision "Smart Hold" peut déclencher CLR) |

**Input :** Order Shopify complète (customer, billing/shipping address, payment method, IP, user agent, cart content, order number), historical orders du customer (Mem0), blacklist cross-merchant (`blacklist.py`), rules config du merchant, ML model output (LightGBM).

**Output :**
```json
{
  "order_id": "gid://shopify/Order/5234567890",
  "score": 82,
  "risk_level": "high",
  "tier_1_blocking_rules_matched": [],
  "tier_2_scoring_rules_matched": [
    { "rule": "freight_forwarder_address", "points": 30 },
    { "rule": "amex_high_value", "points": 15 },
    { "rule": "first_time_customer_large_cart", "points": 10 }
  ],
  "tier_3_ml_score": 0.78,
  "tier_3_top_features": ["email_domain_recent", "ip_country_mismatch_billing"],
  "cross_order_patterns": [
    { "pattern": "same_ip_3_orders_60min", "other_order_ids": ["...", "..."] }
  ],
  "recommendation": "smart_hold",
  "scored_at": "2026-04-24T09:12:03Z"
}
```

**Logique :** Architecture **4-tier** (inspirée de Marble + Jube + HeberTU) détaillée dans `docs/FRAUD_RULES.md` :
- **Tier 1 — Blocking rules** : velocity extrême, blacklist match → auto-reject
- **Tier 2 — Scoring rules** : règles pondérées (freight forwarder +30, AMEX +15, etc.)
- **Tier 3 — ML model** : LightGBM classifier (features engineered dans `ml/feature_engineering.py`)
- **Tier 4 — Human review** : si Smart Hold, le merchant tranche

Score final = somme pondérée Tier 2 + Tier 3 ML × 50 (max 100).

**Edge cases :**
- Nouveau merchant < 30j → ML pas encore calibré, on se base à 80% sur Tier 2, 20% sur Tier 3 baseline
- Blacklist hit certain → score = 100 automatique, pas de nuance
- Order en test (Stripe test mode) → exclu du scoring, retourne `test_order: true`

---

### #17 — Smart Hold

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `fraud_scorer.py` (actor `smart_hold.py`) |
| Endpoint | `GET /api/v1/fraud/holds` / `POST /api/v1/fraud/holds/{id}/release` |
| Composant | `dashboard/fraud/HoldQueue.tsx` |
| Cross-LLM Review | Oui, pour holds > 1000$ |

**Input :** Pre-Ship Score (#16), order value, merchant config (seuil de hold configurable, default = 75).

**Output :**
```json
{
  "hold_id": "hold_xyz",
  "order_id": "gid://shopify/Order/5234567890",
  "order_value_cents": 148000,
  "score": 82,
  "held_at": "2026-04-24T09:12:30Z",
  "reason": "High risk score, high order value",
  "cross_llm_reviewed": true,
  "reviewer_consensus": true,
  "status": "pending_merchant_review",
  "suggested_action": "contact_customer_for_verification",
  "deadline_hours": 24
}
```

**Logique :** Order avec score > 75 ET valeur > 0$ → auto-hold. **Décision affichée au merchant, jamais exécutée automatiquement sur Shopify (pas de cancel order via API).** Pour holds > 1000$, cross-LLM review valide la décision avant de notifier le merchant (évite d'alarmer sur un faux positif cher). Après 24h sans action merchant → rappel push.

**Edge cases :**
- Safe orders (score < 40) → passent sans friction, aucune notification (point de différenciation vs NoFraud)
- Order partiellement payé (split) → hold s'applique à l'ordre entier
- Merchant en vacances (configuré) → hold + notification à l'email backup

---

### #18 — Auto-Evidence Builder (avec approbation humaine obligatoire)

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Chargeback Specialist |
| Analyzer | N/A (actor `evidence_builder.py`) |
| Endpoint | `POST /api/v1/chargebacks/{id}/evidence/generate` / `approve` / `reject` |
| Composant | `dashboard/chargebacks/EvidenceViewer.tsx` + `ApprovalFlow.tsx` |
| Cross-LLM Review | Oui (toujours, quelle que soit la valeur) |

**Input :** Dispute Stripe (reason code, amount, deadline), order complète Shopify (billing, shipping, tracking, items), customer communications (Shopify notes + Klaviyo si connecté), screenshots checkout (si session rec connectée), 3DS data (ECI indicator), tracking GPS (UPS/FedEx API).

**Output :** PDF 10-20 pages structuré selon format Visa/Mastercard (reason code matching), cover letter (rebuttal letter), evidence tabs (delivery proof, communication, AVS/CVV match, 3DS, refund policy acknowledged). Détails dans `docs/CHARGEBACK_WORKFLOW.md`.

**Logique — workflow obligatoire (différenciation #1 vs Chargeflow) :**
1. Chargeback reçu via webhook Stripe → trigger
2. Claude Opus génère draft evidence + rebuttal letter
3. **Cross-LLM review (GPT-5 + Gemini)** : vérifient la cohérence, flag les faiblesses
4. **Merchant notifié**, voit le draft dans l'UI avec : preview PDF, forces/faiblesses du dossier, reason code explained, win probability estimate
5. Merchant **APPROUVE ou REJETTE** — sans approbation explicite, RIEN n'est soumis à Stripe
6. Si approuvé → submission via Stripe Disputes API (`stripe.Dispute.update(evidence=...)`)
7. Tracking de la décision du bank via webhook `dispute.closed`

**Edge cases :**
- Deadline < 48h → alerte haute priorité au merchant (Proactive Evidence Reminder #28)
- Evidence incomplète (ex: pas de tracking disponible) → le draft explicite ce qui manque, merchant peut compléter manuellement avant approbation
- Chargeback sur order bot détecté → draft expose le pattern, high win probability
- Cross-LLM disagreement → escalade visuelle au merchant avec les deux avis

---

### #19 — Email "menace polie" auto

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Chargeback Specialist |
| Analyzer | N/A (actor `polite_threat.py`) |
| Endpoint | `POST /api/v1/chargebacks/{id}/polite-threat/send` |
| Composant | `dashboard/chargebacks/PoliteThreatPreview.tsx` |
| Cross-LLM Review | Oui |

**Input :** Chargeback détails, customer email, template validé (inspiré du template 6-upvotes du thread Reddit 230 upvotes).

**Output :** Email envoyé via Resend au customer chargebacker. Template :
> "We noticed a chargeback was filed against your order #{order_number}. This is likely a misunderstanding — our records show the order was delivered on {delivery_date} at {delivery_address}. If this was a mistake on your or your bank's end, please withdraw the chargeback within 7 days. Otherwise, we will be required to submit evidence and, if the chargeback is ruled fraudulent, send the matter to collections and potentially file an IC3 FBI complaint."

**Logique :** Le template est **proposé** au merchant, jamais envoyé automatiquement sans approbation (cohérent avec la philosophie "merchant approuve tout"). Cross-LLM review valide le ton (ni trop menaçant, ni trop mou). Le merchant peut éditer avant envoi.

**Edge cases :**
- Customer email invalide / bounce → feedback au merchant, suggestion de contacter via téléphone
- Chargeback déjà rétracté → email non envoyé, statut mis à jour
- Customer VIP (LTV > 10K$) → suggestion Claude "Consider a different approach — this is a high-value customer"

---

### #20 — Intégration recouvrement TSI/Rocket

| Champ | Valeur |
|-------|--------|
| Plan | Enterprise |
| Phase | M2 (post-launch) |
| Agent | Chargeback Specialist |
| Analyzer | N/A (service `tsi_recovery.py`) |
| Endpoint | `POST /api/v1/chargebacks/{id}/recovery/send` |
| Composant | `dashboard/chargebacks/RecoveryAction.tsx` |
| Cross-LLM Review | Non (simple API call after approval) |

**Input :** Chargeback lost (merchant a perdu la dispute), tous les documents pré-compilés (evidence, emails, tracking, communication).

**Output :** Dossier envoyé à TSI ou Rocket via leur API, tracking number de la case, statut.

**Logique :** Thread Reddit 230 upvotes : "on envoie CHAQUE chargeback perdu en recouvrement. TSI récupère chaque centime." Bouton "Envoyer en recouvrement" en 1 clic. PDF pré-rempli.

**Edge cases :**
- Chargeback < seuil économique (typiquement < 30$) → TSI refuse, alerte le merchant
- Recovery success → revenue récupéré crédité au P&L avec flag "recovered"
- Recovery échoue → loggué dans history, pattern analysis Mem0

---

### #21 — Plainte IC3 FBI pré-remplie

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M2 |
| Agent | Chargeback Specialist |
| Analyzer | N/A (actor `ic3_filing.py`) |
| Endpoint | `POST /api/v1/chargebacks/{id}/ic3/prefill` |
| Composant | `dashboard/chargebacks/IC3Prefill.tsx` |
| Cross-LLM Review | Oui |

**Input :** Order details, customer data, chargeback reason, evidence bundle.

**Output :** Formulaire IC3 pré-rempli (PDF + data structurée selon champs IC3), prêt à être soumis par le merchant sur https://www.ic3.gov.

**Logique :** Thread 230 upvotes (35 upvotes) : "la plupart voudront arranger ça avant qu'une plainte pénale ne soit déposée". Le simple fait d'avoir la plainte pré-remplie et de la **mentionner** dans l'email #19 a un effet dissuasif. L'envoi réel reste à la discrétion du merchant (on ne peut pas soumettre à l'IC3 en son nom — question légale).

**Edge cases :**
- Customer non-US → IC3 pas applicable, alternative Europol / ActionFraud UK suggérée
- Chargeback de faible montant → Claude recommande "IC3 est réservé aux cas de vraie fraude, envisager plutôt small claims court"

---

### #22 — Ratio Monitor temps réel (Visa/Mastercard VAMP)

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 (CRITIQUE — le merchant peut se faire bannir Stripe) |
| Agent | Fraud Investigator |
| Analyzer | `ratio_monitor.py` |
| Endpoint | `GET /api/v1/fraud/ratio` |
| Composant | `dashboard/fraud/RatioMonitor.tsx` |
| Cross-LLM Review | Oui (pour alertes imminentes du seuil) |

**Input :** Orders count + disputes count sur rolling windows (30j, 60j, 90j), thresholds par card network.

**Output :**
```json
{
  "visa_ratio_30d": 0.72,
  "visa_threshold_monitoring": 0.65,
  "visa_threshold_fines": 0.9,
  "mastercard_ratio_30d": 0.58,
  "mastercard_threshold_monitoring": 0.75,
  "mastercard_threshold_fines": 1.0,
  "status": "visa_monitoring_program",
  "projected_30d": 0.84,
  "alert": {
    "severity": "critical",
    "message": "Visa ratio approaching fines threshold (0.9%). Projected to hit in 12 days at current pace."
  }
}
```

**Logique :** Thread 81 upvotes : merchant banni pour 6 chargebacks. "Dépasser 0.9% = amendes 25K-100K$/mois". Alerte AVANT le seuil (à 80% du seuil), pas après. Projection linéaire sur 30j. Cross-LLM review valide les alertes imminentes pour éviter les fausses alarmes qui créent du stress inutile.

**Edge cases :**
- Stripe "dispute rate" (leur calcul) vs Visa "dispute ratio" (standard network) → on track les 2, on alerte sur le plus strict
- Nouveau store avec peu d'orders (< 100/mois) → volatilité élevée, seuils adaptés
- Chargebacks en attente de verdict → tracked avec flag `pending`

---

### #23 — Blacklist partagée cross-merchants (tokenisée)

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `fraud_scorer.py` (via service `blacklist.py`) |
| Endpoint | Interne (consulté par `fraud_scorer.py`, pas exposé publiquement) |
| Composant | Pas de UI dédiée (surface dans Pre-Ship Score reasons) |
| Cross-LLM Review | Non |

**Input :** Hash tokenisé (salt : `BLACKLIST_HASH_SALT`) de : email, IP, address fingerprint, phone, card fingerprint (via Stripe).

**Output :** Pour une order, retourne `{ blacklist_hits: [{ type: "email", confidence: 0.95, reported_by_merchants_count: 12 }] }`.

**Logique — network effect :** Chaque merchant contribue anonymement à la blacklist en marquant des fraudeurs confirmés. Les hashes sont salés pour empêcher la reverse-engineer. Plus on a de merchants, plus la blacklist protège chacun. Thread 230 upvotes (96 upvotes) : "une blacklist des rétrofacturations, c'est une super idée".

**Edge cases :**
- Faux positif (customer honnête par erreur blacklisté) → appel process merchant → dispute entry → si valide, removal + ajustement confidence
- GDPR/CCPA compliance → seuls des hashes tokenisés, jamais de PII en clair
- Merchant nouveau (< 30j) → peut lire la blacklist mais peut pas contribuer (évite le gaming)

---

### #24 — Friendly Fraud Detector

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `friendly_fraud.py` |
| Endpoint | `GET /api/v1/fraud/friendly-fraud/patterns` |
| Composant | `dashboard/fraud/FriendlyFraudPatterns.tsx` |
| Cross-LLM Review | Non |

**Input :** Chargebacks history, delivery confirmation data (carrier API), customer support logs (Gorgias/Zendesk si connecté), order → dispute timing.

**Output :**
```json
{
  "patterns_detected": [
    {
      "customer_email_hash": "abc123",
      "signal": "delivered_no_support_contact_then_chargeback",
      "orders_count": 3,
      "chargebacks_count": 3,
      "avg_days_delivery_to_chargeback": 18,
      "confidence": 0.92
    }
  ],
  "total_friendly_fraud_cost_30d_cents": 245000
}
```

**Logique :** Thread 230 upvotes : "ils reçoivent le produit, ne contactent JAMAIS le support, et font un chargeback". Pattern = livraison confirmée + zéro interaction customer service + chargeback avec reason "item not received" ou "not as described".

**Edge cases :**
- Customer avec vrai problème (commande endommagée) qui fait chargeback sans contacter → risque faux positif, humain tranche
- Chargeback pour "fraudulent" (reason code 10.4) → pas friendly fraud, vrai fraud, skip

---

### #25 — Billing Descriptor Checker

| Champ | Valeur |
|-------|--------|
| Plan | Free |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `processor_health.py` (sous-routine) |
| Endpoint | `GET /api/v1/fraud/billing-descriptor/health` |
| Composant | `dashboard/fraud/BillingDescriptorCheck.tsx` |
| Cross-LLM Review | Non |

**Input :** Stripe account billing descriptor, shop name Shopify, chargebacks past 90j avec reason "unrecognized".

**Output :**
```json
{
  "current_descriptor": "STRIPE*ACME",
  "shop_name": "Acme Coffee Co.",
  "recognition_score": 0.32,
  "unrecognized_chargebacks_90d": 8,
  "recommendation": "Change descriptor to 'ACMECOFFEE.COM' for better recognition. Expected reduction of unrecognized chargebacks: ~60%.",
  "change_url": "https://dashboard.stripe.com/settings/account"
}
```

**Logique :** Thread chargebacks : clients font opposition car nom non reconnu. Check alignement entre descriptor Stripe et brand, suggère optimisation. Recognition score basé sur : % de chargebacks "unrecognized" vs total chargebacks.

**Edge cases :**
- Merchant multi-brand → descriptor par brand suggéré
- Statement descriptor avec 22 chars limite (Stripe) → alerte si nom tronqué

---

### #26 — Payment Processor Health Dashboard

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `processor_health.py` |
| Endpoint | `GET /api/v1/fraud/processor-health` |
| Composant | `dashboard/fraud/ProcessorHealth.tsx` |
| Cross-LLM Review | Non |

**Input :** Orders + chargebacks breakdown par card brand (Visa / MC / Amex / Discover) et par processor (Stripe / Shop Pay / PayPal).

**Output :**
```json
{
  "by_card_brand": [
    { "brand": "amex", "orders_count": 48, "chargebacks_count": 4, "chargeback_rate_pct": 8.33, "risk_rating": "very_high" },
    { "brand": "visa", "orders_count": 285, "chargebacks_count": 3, "chargeback_rate_pct": 1.05 },
    { "brand": "mastercard", "orders_count": 201, "chargebacks_count": 1, "chargeback_rate_pct": 0.5 },
    { "brand": "discover", "orders_count": 38, "chargebacks_count": 0 }
  ],
  "recommendations": [
    "AMEX chargeback rate is 8.33% (8× industry avg). Consider requiring 3DS for AMEX > $200."
  ]
}
```

**Logique :** Thread processeurs : "Stripe est le pire. Amex encore pire. Mastercard/Discover les meilleurs." On expose les vraies stats par brand pour que le merchant prenne des décisions informées (ex: désactiver AMEX, ajouter 3DS).

**Edge cases :**
- Data < 100 orders par brand → pas de ratio stable, warning
- PayPal chargebacks → logique différente (claim vs dispute), traité à part

---

### #27 — Revenue Dashboard (ROI Anti-Fraude visible)

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | Agrégation des autres analyzers |
| Endpoint | `GET /api/v1/fraud/roi-dashboard` |
| Composant | `dashboard/fraud/AntiFraudROI.tsx` |
| Cross-LLM Review | Non |

**Output :**
```json
{
  "period": "since_install",
  "chargebacks_prevented_count": 18,
  "chargebacks_prevented_value_cents": 245000,
  "chargebacks_won_count": 5,
  "chargebacks_won_value_cents": 78000,
  "total_saved_cents": 323000,
  "false_positive_cost_cents": 42000,
  "net_savings_cents": 281000,
  "pilotprofit_subscription_cost_cents": 12900,
  "roi_multiple": 21.8,
  "win_rate_before_pilotprofit_pct": null,
  "win_rate_after_pilotprofit_pct": 67.0
}
```

**Logique :** Le merchant voit clairement ce qu'il a gagné grâce à PilotProfit (vs Chargeflow review 4+ : "we lost ALL chargebacks since implementing"). ROI clair → renewal facile.

**Edge cases :**
- Install < 30 jours → "data insuffisante" affiché avec progress bar
- Merchant sans chargeback historique → ROI basé uniquement sur False Positive tracker

---

### #28 — Proactive Evidence Reminder

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Chargeback Specialist |
| Analyzer | N/A (Celery task `chargeback_tasks::deadline_reminder`) |
| Endpoint | Notifications via push/email, pas d'endpoint dédié |
| Composant | `dashboard/shared/DeadlineAlertBanner.tsx` |
| Cross-LLM Review | Non |

**Input :** Chargebacks en cours, deadline Stripe, statut draft evidence.

**Output :** Push + email 48h, 24h, 2h avant deadline si evidence pas approvée par merchant.

**Logique :** Reviews Chargeflow (3+) : "submitted before evidence window closed so my proof was never included". Notre reminder est escalant : 48h = info, 24h = warning, 2h = critical avec possibilité "submit draft as-is" si merchant injoignable.

**Edge cases :**
- Weekend/férié → reminder avancé
- Merchant en vacance mode → reminder à l'email backup

---

### #29 — Business Model Adapter

| Champ | Valeur |
|-------|--------|
| Plan | Free (onboarding) |
| Phase | M1 |
| Agent | Fraud Investigator + Chargeback Specialist |
| Analyzer | N/A (config merchant) |
| Endpoint | `POST /api/v1/integrations/merchant/business-model` |
| Composant | `onboarding/BusinessModelSelector.tsx` |
| Cross-LLM Review | Non |

**Input :** Merchant choisit à l'onboarding : physical / digital / dropshipping / subscription / print-on-demand / hybrid.

**Output :** Stratégie adaptée (rules Tier 1 & 2 du fraud scoring), evidence templates adaptés.

**Logique :** Review Chargeflow : "We have a business model that Chargeflow may not be familiar with". Chaque business model a ses patterns de fraude et d'evidence spécifiques :
- **Digital goods** : focus login/download logs, pas de tracking physique
- **Dropshipping** : delay extended, evidence = 3PL tracking
- **Subscription** : focus billing consent, cancellation attempts
- **POD** : focus customization proof

**Edge cases :**
- Merchant hybrid → multi-model actif, règles priorisées par % de revenue
- Changement de business model → recalibration ML + alerte

---

### #30 — AMEX Risk Alert

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `fraud_scorer.py` (règle Tier 2) |
| Endpoint | Surface dans #16 Pre-Ship Score reasons |
| Composant | Intégré dans `RiskScore.tsx` |
| Cross-LLM Review | Non |

**Logique :** Thread 53 upvotes : "On ne prend plus AMEX. J'ai perdu tous les bénéfices du mois à cause d'un seul chargeback AMEX." Card brand AMEX → +15 points au score de risque (configurable). Si AMEX + order > 500$ → +25 points.

**Input/Output/Edge cases :** voir #16.

---

### #31 — Freight Forwarder Detection

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `fraud_scorer.py` (règle Tier 2, via service externe address validator) |
| Endpoint | Surface dans #16 Pre-Ship Score reasons |
| Composant | Intégré dans `RiskScore.tsx` |
| Cross-LLM Review | Non |

**Logique :** Thread $4200 (125 upvotes) : "On n'envoie plus aux commissionnaires de transport." Base de données connue de 10,000+ adresses de freight forwarders (Miami, Delaware, Oregon pour exports). Match sur address fingerprint → **+30 points** au score.

**Input :** Shipping address order, database forwarders (statique + crowd-sourced via merchants).
**Output :** Reason dans Pre-Ship Score : `{ "rule": "freight_forwarder_address", "points": 30, "forwarder_name": "MyUS Miami" }`.

**Edge cases :**
- Adresse ambiguë (pourrait être un particulier) → +10 au lieu de +30, flag "uncertain"
- Merchant confirme "c'est OK pour mon business" → règle désactivée pour ce merchant

---

### #32 — Refund vs Contest Calculator

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Chargeback Specialist |
| Analyzer | N/A (logique dans UI + service) |
| Endpoint | `POST /api/v1/chargebacks/{id}/refund-vs-contest` |
| Composant | `dashboard/chargebacks/RefundVsContestCalc.tsx` |
| Cross-LLM Review | Non |

**Input :** Chargeback amount, Stripe dispute fee (15$), win rate estimé pour ce type de dispute, time cost estimé merchant (heures × taux horaire configuré).

**Output :**
```json
{
  "chargeback_amount_cents": 8500,
  "if_refund": {
    "cost_cents": 8500,
    "outcome": "certain"
  },
  "if_contest": {
    "dispute_fee_cents": 1500,
    "win_probability": 0.45,
    "expected_loss_if_lose_cents": 10000,
    "expected_cost_cents": 7075,
    "time_cost_estimate_hours": 0.5
  },
  "recommendation": "contest",
  "reasoning": "Expected cost of contesting ($70.75) < refunding ($85), despite 45% win rate. Evidence is strong."
}
```

**Logique :** Thread 53 upvotes : "Si quelqu'un n'est pas content, on lui donne une étiquette retour." Parfois rembourser est plus rentable (petit montant, faible win rate). On calcule, on recommande, le merchant décide.

**Edge cases :**
- Win rate unknown (premier chargeback de ce type) → merchant voit "uncertain", recommendation = "contest if evidence strong"

---

### #33 — 3DS Recommender

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `fraud_scorer.py` (enrichi) |
| Endpoint | Surface dans #16 Pre-Ship Score + suggestion UI |
| Composant | `dashboard/fraud/ThreeDSRecommender.tsx` |
| Cross-LLM Review | Non |

**Logique :** Thread $4200 : "Une fois 3DS activé, la responsabilité passe à l'utilisateur." Pour orders high-risk (score > 60), le système suggère au merchant d'activer 3DS via Shopify Checkout Extensibility ou Stripe Radar rule. Liability shift = si le client fait un chargeback fraud après 3DS, la banque paie, pas le merchant.

**Input :** Pre-Ship Score, merchant 3DS config actuelle.
**Output :** Recommandation dans l'UI "Enable 3DS for this order threshold" + lien vers config Shopify.

**Edge cases :**
- Merchant déjà en 3DS 100% → pas de recommandation
- 3DS cause friction → tracking du cart abandonment rate pré/post 3DS pour le merchant

---

### #34 — Card Testing Detector (exclusivité vs Shopify natif)

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `card_testing.py` |
| Endpoint | `GET /api/v1/fraud/card-testing/alerts` |
| Composant | `dashboard/fraud/CardTestingAlert.tsx` |
| Cross-LLM Review | Oui (attaques détectées → cross-LLM valide l'ampleur) |

**Input :** Sliding window des 15 dernières minutes : orders tentés (y compris échecs Stripe), IP sources, card fingerprints, cart values, success/failure rates.

**Output :**
```json
{
  "alert_id": "cardtest_abc",
  "detected_at": "2026-04-24T09:12:00Z",
  "severity": "critical",
  "pattern": "sequential_micro_transactions",
  "indicators": {
    "failed_payments_last_15min": 47,
    "success_rate_pct": 8,
    "distinct_cards_tested": 42,
    "common_ip_ranges": ["185.220.101.0/24"],
    "cart_value_range_cents": [100, 500]
  },
  "recommended_actions": [
    "Enable Cloudflare Bot Fight Mode",
    "Add Stripe Radar rule: block if failed_attempts_from_ip > 3 in 1h",
    "Temporary CAPTCHA on checkout"
  ],
  "projected_cost_if_unchecked_cents": 2500000
}
```

**Logique :** Thread 40 upvotes : "Store fermé 2 ans. $25K+ en frais potentiels." Détection temps réel des patterns typiques :
- Séquence rapide d'orders avec cards différentes mais même IP
- Ratio failed/success anormal (> 70% failed sur 15 min)
- Cart values très bas ($1-5) pour "tester" la validité de la carte

Cross-LLM review pour valider que c'est une vraie attaque (pas un marketing push légitime qui génère beaucoup de traffic).

**Edge cases :**
- BFCM / flash sale → baseline ajustée, seuils plus hauts
- Merchant très petit (< 10 orders/jour) → seuils plus bas, sensibilité élevée
- False positive (store qui a juste beaucoup de trial cards légitimes) → merchant peut whitelister

---

# MODULE 3 : INTELLIGENCE (5 features)

---

### #35 — Total Ad Spend Tracker 100% (alerte écart > 10%)

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Ads Sync + Data Integrity |
| Analyzer | `ad_spend.py` + `data_integrity.py` |
| Endpoint | Surface dans `/api/v1/integrity/alerts` |
| Composant | `dashboard/ads/SpendDiscrepancyAlert.tsx` |
| Cross-LLM Review | Non |

**Logique :** Duplicata intentionnel avec #5 — la feature est si critique qu'elle a son propre scanner dédié qui alerte si l'écart entre ad spend **rapporté dans Shopify UTM** et ad spend **réel via API Meta/Google/TikTok** dépasse 10%. Faille BeProfit exploitée frontalement.

**Input/Output/Edge cases :** voir #5 et #10.

---

### #36 — Smart Alerts Temps Réel (unified feed)

| Champ | Valeur |
|-------|--------|
| Plan | Starter |
| Phase | M1 |
| Agent | Supervisor (agrège les alertes de tous les spécialistes) |
| Analyzer | N/A (agrégation) |
| Endpoint | `GET /api/v1/notifications` (unified feed) |
| Composant | `dashboard/shared/AlertFeed.tsx` |
| Cross-LLM Review | Non |

**Output :**
```json
{
  "alerts": [
    {
      "id": "alert_1",
      "type": "profit_drop",
      "severity": "warning",
      "agent": "profit_analyst",
      "message": "Gross margin on TSHIRT-BLK-L dropped 23% this week",
      "created_at": "..."
    },
    {
      "id": "alert_2",
      "type": "chargeback_spike",
      "severity": "critical",
      "agent": "fraud_investigator",
      "message": "3 chargebacks in the last 4 hours — projected ratio: 1.2%",
      "created_at": "..."
    }
  ]
}
```

**Logique :** GoProfit : "catches profit drops & spikes the moment they happen." On fait ça + fraud + chargebacks + refunds + integration health dans le **même flux**. Push immédiat via notification Shopify admin + web-push + email (selon prefs).

**Edge cases :**
- Alert storm (10+ alertes en 5min) → groupées dans un "digest" pour éviter le spam
- Merchant en DND (do not disturb mode) → seulement critical passent

---

### #37 — False Positive Cost Tracker (exclusivité)

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | Extension de `fraud_scorer.py` avec feedback loop |
| Endpoint | `GET /api/v1/fraud/false-positive-cost` |
| Composant | `dashboard/fraud/FalsePositiveCard.tsx` |
| Cross-LLM Review | Non |

**Output :**
```json
{
  "period": "last_30d",
  "orders_held": 42,
  "confirmed_fraud_count": 28,
  "confirmed_fraud_saved_cents": 485000,
  "false_positives_count": 14,
  "false_positives_lost_cents": 178000,
  "net_benefit_cents": 307000,
  "accuracy_pct": 66.7,
  "trend": "improving",
  "recommendation": "Consider slightly lowering hold threshold from 75 to 72 to reduce false positives"
}
```

**Logique :** NoFraud/Wyllo review : "too sensitive, cancelling genuine orders. Probably costing me more in lost orders than fraud orders." Quand Smart Hold bloque une order :
- Si merchant release (approve) → on tag `false_positive` après 45j (période où un chargeback aurait pu apparaître)
- Si merchant cancel → on tag `confirmed_fraud` si blacklist match ultérieur ou indicators forts

Après 30 jours, on expose le ratio. Feedback loop : ces données re-entraînent le ML model.

**Edge cases :**
- Order hold + release + chargeback ultérieur → re-classifié en `correct_block_released_by_mistake`
- Period < 45j → data "preliminary"

---

### #38 — Promo Planner / Simulator

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M2 (post-launch) |
| Agent | Profit Analyst + Fraud Investigator |
| Analyzer | `cashflow.py` + `fraud_scorer.py` (simulation mode) |
| Endpoint | `POST /api/v1/profit/promo/simulate` |
| Composant | `dashboard/profit/PromoPlanner.tsx` |
| Cross-LLM Review | Oui (impact financier important) |

**Input :** Promo config (discount %, durée, SKUs concernés, codes, audience), historical data saisonnière.

**Output :**
```json
{
  "simulation": {
    "discount_pct": 20,
    "duration_days": 3,
    "affected_skus_count": 12
  },
  "projected": {
    "revenue_lift_cents": 450000,
    "profit_impact_cents": -120000,
    "profit_impact_pct": -8.5,
    "ad_spend_required_cents": 80000,
    "chargeback_risk_increase_pct": 12,
    "expected_fraud_orders": 2
  },
  "recommendation": "Run promo, but enable 3DS for new customers during this window to mitigate fraud risk."
}
```

**Logique :** GoProfit fait la partie profit. On ajoute **le risque fraude combiné** (les promos agressives attirent fraudeurs et card testing). Claude Opus génère la recommandation finale, validée par cross-LLM.

**Edge cases :**
- Pas assez de promos passées → simulation dégradée, warning
- Produits en bundle → recalcul inter-SKU

---

### #39 — Return Abuse Detector

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Fraud Investigator |
| Analyzer | `return_abuse.py` |
| Endpoint | `GET /api/v1/fraud/return-abuse` |
| Composant | `dashboard/fraud/ReturnAbuseList.tsx` |
| Cross-LLM Review | Non |

**Output :**
```json
{
  "abusers": [
    {
      "customer_email_hash": "def456",
      "orders_6m": 7,
      "returns_6m": 5,
      "return_rate_pct": 71.4,
      "pattern": "wardrobing",
      "financial_impact_cents": -34000,
      "suggestion": "Flag for manual review on next order, or limit return window"
    }
  ],
  "total_abuse_cost_30d_cents": 89000
}
```

**Logique :** NoFraud/Wyllo fait "policy abuse" mais sans lien au profit. Nous : pattern = > 3 retours en 6 mois + corrélation avec comportement d'achat (porté et retourné = wardrobing). Coût réel incluant return processing (from #6).

**Edge cases :**
- Customer B2B avec returns légitimes élevés → merchant peut whitelister
- Seasonal returners → Mem0 apprend

---

# MODULE 4 : TARIFS & LANDED COST (3 features)

---

### #40 — Landed Cost Calculator

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `landed_cost.py` |
| Endpoint | `GET /api/v1/tariffs/landed-cost/{sku}` |
| Composant | `dashboard/tariffs/LandedCostTable.tsx` |
| Cross-LLM Review | Non |

**Input :** Product COGS (merchant-declared), country of origin, HTS code (merchant-declared ou inféré par Claude), shipping cost from origin, current tariff rate (API US Customs & Border Protection ou équivalent), destination country.

**Output :**
```json
{
  "sku": "TSHIRT-BLK-L",
  "cogs_cents": 1200,
  "origin_country": "CN",
  "hts_code": "6109.10.00",
  "destination_country": "US",
  "tariff_rate_pct": 12,
  "tariff_amount_cents": 144,
  "shipping_from_origin_cents": 360,
  "landed_cost_cents": 1704,
  "retail_price_cents": 3500,
  "perceived_margin_pct": 65.7,
  "real_margin_pct": 51.3,
  "margin_erosion_pct": 14.4
}
```

**Logique :** Reddit r/smallbusiness (156 upvotes) : "This requires SKU-level margin visibility." Le merchant vend un t-shirt 35$, pense faire 48% de marge (retail - COGS), mais la vraie marge landed-cost est 23% après tariff + shipping + duties. Aucun profit tracker Shopify ne fait ça en avril 2026.

**Edge cases :**
- HTS code manquant → Claude Opus infère à partir du product description + catégorie Shopify, merchant valide
- Tariff qui change → recalcul automatique des marges, alerte Proactive Bug (#11)
- Free Trade Agreement applicable (ex: USMCA) → tariff 0%

---

### #41 — Tariff Impact Simulator

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `tariff_impact.py` |
| Endpoint | `POST /api/v1/tariffs/simulate` |
| Composant | `dashboard/tariffs/TariffSimulator.tsx` |
| Cross-LLM Review | Oui (recommandations stratégiques) |

**Input :** Scenario (ex: "If China tariffs increase by 25%"), affected SKUs, current sales velocity.

**Output :**
```json
{
  "scenario": "china_tariff_increase_25pct",
  "affected_skus_count": 47,
  "monthly_profit_impact_cents": -420000,
  "per_sku_impact": [ ... ],
  "mitigation_options": [
    { "action": "Raise price by 8%", "profit_recovery_cents": 280000, "expected_sales_drop_pct": 4 },
    { "action": "Diversify supplier to Vietnam", "profit_recovery_cents": 380000, "implementation_time": "3 months" },
    { "action": "Absorb the cost", "profit_recovery_cents": 0, "cash_reserve_needed_months": 5 }
  ],
  "recommendation": "Combination: raise price 5% + start Vietnam supplier exploration. Break-even in 4 weeks."
}
```

**Logique :** US tariffs changent toutes les semaines en 2026. Merchants ne peuvent pas anticiper sans outil. Cross-LLM review pour les recommandations (Claude + GPT-5 + Gemini cross-check).

**Edge cases :**
- Tariff historique cassant (ex: +100%) → scenarios extrêmes gérés avec warning "unprecedented"
- Scenario "tariff suppression" (0%) → merchant informé du potentiel gain + risque si rollback

---

### #42 — Non-Refundable Duty Tracker

> **Note :** feature numérotée #42 alors que le module annonce 3 features → #42 est bien la 3e du module (les #40 et #41 précèdent). Total module 4 = 3 features, total projet = 41 features (#1-#41 hors renumbering). Voir table récap en fin.

**Correction numérotation :**

### #41 (final) — Non-Refundable Duty Tracker

| Champ | Valeur |
|-------|--------|
| Plan | Pro |
| Phase | M1 |
| Agent | Profit Analyst |
| Analyzer | `return_cost.py` + `landed_cost.py` (intersection) |
| Endpoint | `GET /api/v1/tariffs/duties-lost` |
| Composant | `dashboard/tariffs/DutiesLostCard.tsx` |
| Cross-LLM Review | Non |

**Input :** Refunded orders with import duties (via #40 landed cost data), return destination (domestic return = duty lost, return to origin = duty potentially recoverable via drawback).

**Output :**
```json
{
  "period": "last_30d",
  "returns_with_duty_count": 18,
  "total_duty_paid_cents": 2600,
  "recoverable_via_drawback_cents": 0,
  "lost_duty_cents": 2600,
  "hidden_cost_per_return_avg_cents": 144,
  "annualized_projection_cents": 31200
}
```

**Logique :** Nventory.io : "Import duties are non-refundable when products are returned." Pour chaque retour, on track le duty payé à l'origine, maintenant perdu. Permet de flagger pour les SKUs high-return, la vraie rentabilité.

**Edge cases :**
- Drawback US possible (reexport) → `recoverable_via_drawback_cents` > 0, suggestion process
- Commingled import (DDU vs DDP) → duty déjà payé par customer, on track différemment

---

# RÉCAPITULATIF — 41 FEATURES

| # | Feature | Module | Plan | Agent |
|---|---------|--------|------|-------|
| 1 | True Profit | Profit | Free | Profit Analyst |
| 2 | App Cost Allocator | Profit | Starter | Profit Analyst |
| 3 | Daily P&L | Profit | Free | Profit Analyst |
| 4 | Margin Alerts | Profit | Starter | Profit Analyst |
| 5 | Ad Spend Tracker 100% | Profit | Starter | Ads Sync |
| 6 | Return Cost Calculator | Profit | Starter | Profit Analyst |
| 7 | OPEX Categorizer | Profit | Starter | Profit Analyst |
| 8 | Tax Ready Export | Profit | Starter | Profit Analyst |
| 9 | Cashflow Forecast | Profit | Pro | Profit Analyst |
| 10 | Data Integrity Check | Profit | Free | Data Integrity |
| 11 | Proactive Bug Alert | Profit | Starter | Data Integrity |
| 12 | LTV propre | Profit | Pro | Profit Analyst |
| 13 | Multi-Fulfillment Sync | Profit | Pro | Profit Analyst |
| 14 | Post-Purchase Upsell Tracker | Profit | Starter | Profit Analyst |
| 15 | Currency FX Loss Tracker | Profit | Pro | Profit Analyst |
| 16 | Pre-Ship Score | Anti-Fraude | Free | Fraud Investigator |
| 17 | Smart Hold | Anti-Fraude | Starter | Fraud Investigator |
| 18 | Auto-Evidence Builder | Anti-Fraude | Pro | Chargeback Specialist |
| 19 | Email menace polie | Anti-Fraude | Pro | Chargeback Specialist |
| 20 | Recouvrement TSI/Rocket | Anti-Fraude | Enterprise | Chargeback Specialist |
| 21 | Plainte IC3 FBI | Anti-Fraude | Pro | Chargeback Specialist |
| 22 | Ratio Monitor VAMP | Anti-Fraude | Starter | Fraud Investigator |
| 23 | Blacklist cross-merchants | Anti-Fraude | Pro | Fraud Investigator |
| 24 | Friendly Fraud Detector | Anti-Fraude | Pro | Fraud Investigator |
| 25 | Billing Descriptor Checker | Anti-Fraude | Free | Fraud Investigator |
| 26 | Processor Health Dashboard | Anti-Fraude | Pro | Fraud Investigator |
| 27 | Revenue Dashboard Anti-Fraude | Anti-Fraude | Starter | Fraud Investigator |
| 28 | Proactive Evidence Reminder | Anti-Fraude | Pro | Chargeback Specialist |
| 29 | Business Model Adapter | Anti-Fraude | Free | Fraud + Chargeback |
| 30 | AMEX Risk Alert | Anti-Fraude | Starter | Fraud Investigator |
| 31 | Freight Forwarder Detection | Anti-Fraude | Pro | Fraud Investigator |
| 32 | Refund vs Contest Calculator | Anti-Fraude | Starter | Chargeback Specialist |
| 33 | 3DS Recommender | Anti-Fraude | Pro | Fraud Investigator |
| 34 | Card Testing Detector | Anti-Fraude | Starter | Fraud Investigator |
| 35 | Total Ad Spend Tracker 100% (alerte écart) | Intelligence | Starter | Ads Sync + Data Integrity |
| 36 | Smart Alerts Unified Feed | Intelligence | Starter | Supervisor |
| 37 | False Positive Cost Tracker | Intelligence | Pro | Fraud Investigator |
| 38 | Promo Planner Simulator | Intelligence | Pro | Profit + Fraud |
| 39 | Return Abuse Detector | Intelligence | Pro | Fraud Investigator |
| 40 | Landed Cost Calculator | Tarifs | Pro | Profit Analyst |
| 41 | Tariff Impact Simulator + Non-Refundable Duty Tracker | Tarifs | Pro | Profit Analyst |

**Nota bene :** Les 3 features du Module 4 "Tarifs & Landed Cost" sont : Landed Cost Calculator (#40), Tariff Impact Simulator (intermédiaire), Non-Refundable Duty Tracker (final #41). Dans la doc détaillée plus haut, le simulator et le duty tracker sont consolidés conceptuellement ; en implémentation, ils ont leurs analyzers et endpoints distincts (`tariff_impact.py` vs logique dans `return_cost.py` + `landed_cost.py`).

---

## RÉCAP PAR MODULE ET PAR PLAN

| Module | Features | Free | Starter | Pro | Enterprise |
|--------|----------|------|---------|-----|-----------|
| Profit | 15 | 2 | 8 | 5 | 0 |
| Anti-Fraude | 18 | 3 | 6 | 8 | 1 |
| Intelligence | 5 | 0 | 2 | 3 | 0 |
| Tarifs | 3 | 0 | 0 | 3 | 0 |
| **Total** | **41** | **5** | **16** | **19** | **1** |

---

**Dernière mise à jour :** 2026-04-24
**Status :** Spec figée pour M1 launch. Modifications post-M1 via RFC dans `docs/rfc/`.
