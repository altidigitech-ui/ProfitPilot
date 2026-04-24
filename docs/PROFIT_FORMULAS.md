# PROFIT_FORMULAS.md — Formules math exactes PilotProfit

> **Référence unique pour toutes les formules de calcul financier.**
> **Si une feature Profit a besoin d'une formule, elle est ici. Point.**
> **Aucun analyzer ne doit inventer une formule qui n'est pas dans ce fichier.**

---

## PRINCIPES NON-NÉGOCIABLES

### 1. Types numériques

```python
from decimal import Decimal, ROUND_HALF_EVEN, getcontext

# Précision fixée pour tout le module profit
getcontext().prec = 28
getcontext().rounding = ROUND_HALF_EVEN   # Banker's rounding (IEEE 754)

# TOUJOURS Decimal pour les calculs
revenue: Decimal = Decimal("99.99")

# TOUJOURS cents INTEGER pour le stockage DB et les APIs
revenue_cents: int = int(revenue * 100)  # 9999

# JAMAIS float (ROUND_HALF_EVEN + erreurs IEEE 754)
bad: float = 99.99  # INTERDIT
```

**Pourquoi banker's rounding ?** Sur 10 000 transactions à 0,005 $, le rounding "half up" classique crée un biais cumulé de +25 $ en notre faveur (ou contre le merchant). ROUND_HALF_EVEN neutralise ce biais.

### 2. Multi-currency

Toute formule manipule des montants dans UNE SEULE devise à la fois — celle de l'order en question (`order.presentment_currency`).

Si besoin de cross-currency aggregation (P&L global d'un merchant qui vend en USD/EUR/GBP) :
- Chaque ligne est stockée dans sa devise d'origine en DB
- L'agrégation est convertie à la `shop_currency` (devise principale du merchant Shopify) au taux du **jour de l'order** (pas du jour actuel, sinon les chiffres historiques changent rétroactivement)
- Taux FX via `services/fx_rates.py` (provider configuré via `FX_RATES_PROVIDER`)

### 3. Timestamp reference

Toute formule qui dépend d'un taux (FX, tarif, COGS variable) utilise le taux **au jour de création de l'order** (`order.created_at`). Pas au jour de capture, pas au jour d'aujourd'hui.

### 4. Data Integrity avant calcul

Aucune formule ne s'exécute si `data_integrity_check(sources=["shopify", "stripe"])` renvoie `variance_pct > 2%` sur les inputs requis. Dans ce cas → `ProfitError(code=DATA_INTEGRITY_FAILED)`. Voir `docs/DATA_INTEGRITY.md`.

---

## 1. TRUE PROFIT (feature #1)

### 1.1 Gross Profit

```
gross_profit = revenue
             - cogs
             - shipping_cost_merchant
             - payment_fees
```

Où :
- `revenue` = `order.subtotal_price` (après discounts, avant tax et shipping)
- `cogs` = somme des `line_item.quantity × line_item.unit_cost` (voir section 1.3 pour COGS)
- `shipping_cost_merchant` = **coût réel** du shipping payé par le merchant (via ShipStation/ShipBob/MCF API), PAS le shipping facturé au customer
- `payment_fees` = Stripe fees réels (via Stripe Balance Transactions API, pas estimation)

### 1.2 Net Profit

```
net_profit = gross_profit
           - ad_spend_allocated
           - apps_cost_allocated
           - duties
           - refund_cost
           - opex_allocated
```

Où :
- `ad_spend_allocated` = voir section 3 (attribution)
- `apps_cost_allocated` = voir section 4
- `duties` = voir section 8 (landed cost)
- `refund_cost` = voir section 6 (return cost)
- `opex_allocated` = part d'OPEX allouée à cet order (généralement 0 pour un calcul par-order ; appliqué au niveau daily P&L)

### 1.3 COGS — 3 méthodes supportées

Le merchant choisit sa méthode dans les settings (défaut : **moving average**).

**Méthode A — Shopify native `unit_cost`**
```
line_cogs = line_item.quantity × inventory_item.unit_cost
```
Le merchant a déclaré son unit cost dans Shopify Admin.

**Méthode B — Moving Average Cost (MAC)**
```
moving_avg_cost_t = (moving_avg_cost_{t-1} × inventory_qty_{t-1} + new_batch_cost × new_batch_qty)
                  / (inventory_qty_{t-1} + new_batch_qty)
```
Recalculé à chaque réception de stock (via Shopify Inventory API ou import CSV manuel). Stocké dans `cogs_history` table.

**Méthode C — FIFO (First In, First Out)**
```
pour chaque line_item vendu :
    reste_qty = line_item.quantity
    line_cogs = 0
    pour chaque batch in inventory_batches_fifo (ordre d'arrivée croissant) :
        qty_from_batch = min(batch.remaining_qty, reste_qty)
        line_cogs += qty_from_batch × batch.unit_cost
        batch.remaining_qty -= qty_from_batch
        reste_qty -= qty_from_batch
        si reste_qty == 0 : break
```

LIFO (Last In First Out) **n'est pas supporté** : interdit en IFRS et non-recommandé pour e-commerce.

### 1.4 Multi-fulfillment (feature #13)

Si un order est split entre N fulfillment sources :

```
pour chaque fulfillment in order.fulfillments :
    line_shipping_cost += fulfillment.shipping_cost_real
    line_storage_cost += allocated_storage_cost(fulfillment.location_id, items_qty)
    line_pickpack_cost += fulfillment.pickpack_fee
```

`allocated_storage_cost` = coût mensuel de storage du warehouse × (items_qty / throughput_mensuel_warehouse).

---

## 2. DAILY P&L (feature #3)

### 2.1 Agrégation

```
pour chaque day in period :
    day.revenue = Σ(orders_created_on_day.subtotal_price)  # converted to shop_currency
    day.cogs = Σ(orders_created_on_day.cogs)
    day.shipping = Σ(orders_created_on_day.shipping_cost_merchant)
    day.payment_fees = Σ(orders_created_on_day.payment_fees)
    day.ad_spend = Σ(ad_channels.daily_spend[day])  # de Meta/Google/TikTok APIs
    day.apps_cost = daily_app_cost (voir section 4)
    day.duties = Σ(orders_created_on_day.duties)
    day.opex = daily_opex (voir section 7)
    day.refunds_cost = Σ(refunds_processed_on_day.real_cost)

    day.gross_profit = day.revenue - day.cogs - day.shipping - day.payment_fees
    day.net_profit  = day.gross_profit - day.ad_spend - day.apps_cost - day.duties - day.opex - day.refunds_cost
```

### 2.2 Orders multi-days

Un order créé le jour J avec refund le jour J+5 :
- Revenue comptabilisé le jour J (date de création)
- Refund cost comptabilisé le jour J+5 (date du refund)
- **Pas de réécriture rétroactive** du jour J (sinon les rapports historiques changeraient)

### 2.3 Currency conversion pour aggregation

```
order.revenue_shop_currency = order.subtotal_price × fx_rate(
    from=order.presentment_currency,
    to=shop.currency,
    date=order.created_at.date()   # taux du jour de l'order, pas d'aujourd'hui
)
```

---

## 3. AD SPEND ATTRIBUTION (feature #5, #14)

### 3.1 Règle d'attribution (choisie par merchant)

**Mode A — First-touch channel** (défaut)
```
si order.customer.first_touch_utm_source existe :
    order.channel = order.customer.first_touch_utm_source
sinon si order.landing_site_ref contient "fbclid" :
    order.channel = "meta"
sinon si order.landing_site_ref contient "gclid" :
    order.channel = "google"
sinon si order.landing_site_ref contient "ttclid" :
    order.channel = "tiktok"
sinon :
    order.channel = "direct"
```

**Mode B — Last-touch channel**
Pareil mais basé sur `last_touch_utm_source`.

**Mode C — Linear (N-touch)**
```
order.attribution = { channel: weight_pct, ... }
# ex: { "meta": 40, "google": 35, "direct": 25 }
```

**Mode D — Data-driven (Pro/Enterprise only)**
Basé sur Mem0 historical patterns + ML model. Requiert > 500 orders historiques.

### 3.2 Spend allocation per order

```
channel_spend_day_d = sum(meta_daily_spend[d], google_daily_spend[d], tiktok_daily_spend[d]) par canal
channel_orders_day_d = count(orders) par canal sur le jour d

ad_spend_per_order(channel, day) = channel_spend_day_d[channel] / channel_orders_day_d[channel]
```

Pour un order attribué au canal `meta` le jour d :
```
order.ad_spend_allocated = ad_spend_per_order("meta", d)
```

### 3.3 ROAS effectif

```
channel_roas = revenue_attributed_to_channel / channel_spend
channel_real_profit = revenue_attributed_to_channel - cogs_attributed_to_channel - channel_spend
```

### 3.4 Discrepancy Alert (feature #35)

```
reported_spend_shopify_utm = Σ(orders.utm_source=channel).ad_spend_claimed_by_shopify
actual_spend_api = API.get_total_spend(channel, period)

variance_pct = |actual_spend_api - reported_spend_shopify_utm| / actual_spend_api × 100

si variance_pct > 10 : déclencher alerte
```

Exemple concret (faille BeProfit) :
- Shopify UTM rapporte : 93 000 $ (Google)
- Google Ads API rapporte : 620 000 $
- Variance : 85% → **ALERTE CRITIQUE**

---

## 4. APPS COST ALLOCATOR (feature #2)

### 4.1 Cost per order

```
pour chaque app in shopify_app_subscriptions :
    app.monthly_cost = app.recurring_charge + app.usage_charges_30d_moving
    app.cost_per_order = app.monthly_cost / orders_count_30d_moving
```

### 4.2 Allocation dans True Profit

Pour chaque order :
```
order.apps_cost_allocated = Σ(app.cost_per_order for app in active_apps)
```

### 4.3 Worth-it score

```
pour chaque app dans apps_with_attributable_revenue :
    app.revenue_attributed = Σ(orders.revenue where orders.utm_source or orders.tags contains app.handle)
    app.worth_it_score = app.revenue_attributed / app.monthly_cost
```

`worth_it_score > 10` → très rentable. `< 2` → douteux. `< 1` → app perd de l'argent.

### 4.4 Ghost billing detection (feature #11)

```
pour chaque app in shopify_app_subscriptions :
    si app.status == "uninstalled" AND app.last_charge_date > uninstall_date :
        alert("ghost_billing_detected", app.handle, app.amount, uninstall_date)
```

---

## 5. LTV PROPRE (feature #12)

### 5.1 Eligibility du customer

```
customer_eligible_for_ltv = (
    NOT customer.is_bot_flagged     # marqué par Fraud Investigator
    AND customer.orders_count >= 1  # exclure non-acheteurs (faille TrueProfit)
    AND NOT customer.is_test_customer
    AND NOT customer.email IN bounce_list
)
```

### 5.2 LTV par horizon

```
pour chaque customer in eligible_customers :
    orders_within_horizon = customer.orders.filter(
        created_at BETWEEN customer.first_order_at AND customer.first_order_at + horizon_days
    )
    customer.ltv[horizon] = Σ(order.net_profit for order in orders_within_horizon)
```

### 5.3 Cohort LTV

```
cohort_ltv[cohort][horizon] = mean(customer.ltv[horizon] for customer in cohort)
```

Où `cohort` est défini par `customer.first_order_at.year_quarter` (ex: "2025-Q4").

### 5.4 Repeat Purchase Rate

```
repeat_rate = count(customers with orders_count >= 2) / count(all_eligible_customers)
```

---

## 6. RETURN COST (feature #6)

### 6.1 Real cost formula

```
real_return_cost = product_value_lost
                 + return_shipping_cost
                 + original_shipping_cost_unrecoverable
                 + payment_fee_unrecoverable
                 + duty_unrecoverable
                 + restocking_labor_estimate
                 + dead_stock_loss
```

Détail de chaque composant :

**product_value_lost**
```
si return.disposition == "resellable" : product_value_lost = 0
si return.disposition == "damaged" : product_value_lost = line_cogs × 0.7  # 70% loss avg
si return.disposition == "dead_stock" : product_value_lost = line_cogs     # 100% loss
```

**return_shipping_cost** = coût réel du retour (tracking label fee si merchant-paid, 0 sinon)

**original_shipping_cost_unrecoverable** = shipping initial payé par merchant (Stripe ne refund pas cette part)

**payment_fee_unrecoverable** = Stripe fees sur la transaction initiale (Stripe keep the fee on refunds)
```
payment_fee_unrecoverable = Σ(stripe_balance_transactions.where(type=="refund", refund_id=refund.id).fee)
```

**duty_unrecoverable** = duties payées à l'import, non-refondables (feature #41)
```
si return.destination_country == origin_country :   # reexport → drawback US possible
    duty_unrecoverable = duty × (1 - drawback_recovery_rate)  # 0.99 × duty typically
sinon :
    duty_unrecoverable = duty   # 100% perdu
```

**restocking_labor_estimate**
```
restocking_labor_estimate_cents = warehouse.labor_rate_per_minute × return.restocking_minutes
```
Défaut : 200 cents (2$) par return si merchant n'a pas configuré.

**dead_stock_loss** = si l'item ne peut pas être revendu
```
si produit.is_perishable OR produit.age_days > 365 : dead_stock_loss = line_cogs
sinon : 0
```

### 6.2 Partial refund

```
line_refund_ratio = refund.line_item.quantity / order.line_item.quantity
real_return_cost_partial = real_return_cost × line_refund_ratio
```

---

## 7. CASHFLOW FORECAST (feature #9)

### 7.1 Revenue forecast

**Baseline method (rolling average)**
```
baseline_daily_revenue = mean(revenue_last_N_days) where N = 30
```

**Seasonality multipliers** (par jour de la semaine, par mois)
```
dow_multiplier[dow] = mean(revenue on dow) / baseline_daily_revenue  # sur 90j history
month_multiplier[month] = mean(revenue in month) / baseline_daily_revenue  # sur 365j history

projected_revenue[date] = baseline_daily_revenue
                        × dow_multiplier[date.dow]
                        × month_multiplier[date.month]
```

### 7.2 Cost forecast

```
recurring_costs[date] = Σ(app.monthly_cost / 30) + Σ(team_cost / 30) + Σ(recurring_opex / 30)
variable_costs[date] = projected_revenue[date] × avg_variable_cost_ratio
ad_spend_projected[date] = rolling_mean(ad_spend_last_14d)   # hypothèse de scaling constant
```

### 7.3 Balance projeté

```
balance[t] = balance[t-1] + projected_revenue[t] - projected_costs[t] - upcoming_payables[t]
```

`upcoming_payables` = POs suppliers (Xero/QuickBooks), Stripe payout schedule inversé (si payout sort, cash diminue).

### 7.4 Confidence score

```
confidence[horizon_days] = max(0.3, 0.95 - horizon_days × 0.007)
```

- J+7 : confidence ≈ 0.9
- J+30 : confidence ≈ 0.74
- J+90 : confidence ≈ 0.31

Ajustement volatility :
```
volatility = stddev(revenue_last_30d) / mean(revenue_last_30d)
si volatility > 0.5 : confidence × 0.5
```

---

## 8. LANDED COST (feature #40)

### 8.1 Formule

```
landed_cost = cogs
            + shipping_from_origin
            + tariff
            + import_fees
            + brokerage_fees
            + insurance
```

### 8.2 Tariff computation

```
tariff = declared_customs_value × tariff_rate(hts_code, origin_country, destination_country)
```

Où `tariff_rate` vient de :
- **US** : USITC DataWeb API (Section 301, 232, 122 rates)
- **EU** : TARIC API
- **UK** : Trade Tariff API
- **Autres** : fallback rate merchant-declared

Mis à jour **quotidiennement** via cron (les tariffs US changent toutes les semaines en 2026).

### 8.3 Real margin

```
perceived_margin = (retail_price - cogs) / retail_price × 100
real_margin     = (retail_price - landed_cost) / retail_price × 100
margin_erosion  = perceived_margin - real_margin
```

Exemple :
- Retail : 3500 ¢, COGS : 1200 ¢ → perceived_margin = 65.7%
- Landed : 1704 ¢ → real_margin = 51.3%
- Erosion : 14.4 points

### 8.4 Tariff Impact Simulator (feature #41 inner)

```
scenario_tariff_rate = current_rate + delta_pct
pour chaque sku in affected_skus :
    scenario_landed_cost = sku.cogs + sku.shipping + declared_value × scenario_tariff_rate
    scenario_margin = (sku.retail - scenario_landed_cost) / sku.retail
    scenario_monthly_profit_impact = (scenario_margin - current_margin) × sku.monthly_revenue
total_impact = Σ(sku.scenario_monthly_profit_impact)
```

---

## 9. CURRENCY FX LOSS (feature #15) — exclusivité

### 9.1 Loss par order multi-currency

```
order_fx_loss_usd = order_value_in_shop_currency_at_order_date
                  - settled_value_usd_from_stripe
```

Détail :
```
order_value_in_shop_currency_at_order_date =
    order.presentment_price
    × fx_rate(from=order.presentment_currency, to=shop.currency, date=order.created_at)

settled_value_usd_from_stripe =
    stripe_balance_transaction.amount  # déjà en settlement currency (shop currency)
    - stripe_balance_transaction.exchange_rate_fees
```

### 9.2 Aggregation par currency

```
pour chaque currency in distinct(orders.presentment_currency) :
    currency_loss = Σ(order.fx_loss for order in orders where order.presentment_currency == currency)
    currency_loss_pct = currency_loss / Σ(order.order_value_shop_currency) × 100
```

### 9.3 Recommendation logic

```
si currency.loss_pct > 2.5 :
    recommendation = "Consider Wise Business or multi-currency settlement (save ~1.5%)"
si currency.volatility_30d > 5% :
    recommendation += " Currency volatile — set auto-conversion thresholds"
```

---

## 10. POST-PURCHASE UPSELL (feature #14)

### 10.1 Identification

Un upsell est un order avec :
- `order.source_identifier` contient `"post_purchase"` OU
- `order.tags` contient `"upsell"`, `"post-purchase"`, `"aftersell"` OU
- `order.note_attributes` contient `{name: "upsell_parent_order", value: <order_id>}`

### 10.2 Attribution

```
upsell.parent_order = resolve_from_note_attributes(upsell.note_attributes)
upsell.standalone_profit = compute_true_profit(upsell)   # appliqué avec formulas section 1
```

### 10.3 Acceptance rate

```
acceptance_rate = count(upsell_orders) / count(orders_with_upsell_offered)
```

`orders_with_upsell_offered` = via webhook app d'upsell (ReConvert/AfterSell/Zipify fournissent cet événement).

---

## 11. OPEX ALLOCATION (feature #7)

### 11.1 Catégorisation

Catégories IRS Schedule C compatible (pour export tax #8) :
- `software` (apps, SaaS subscriptions)
- `payment_processing` (Stripe/Shopify fees)
- `advertising` (ad spend)
- `shipping_operations` (3PL, carriers)
- `team` (salaries, freelance, contractors)
- `office` (rent, utilities)
- `professional_services` (legal, accounting)
- `taxes_licenses` (sales tax, business license)
- `uncategorized`

### 11.2 Auto-categorization

```
pour chaque transaction in qbo_xero_imports + stripe_charges + shopify_app_charges :
    categorized = claude_categorize(transaction.description, transaction.merchant_name, mem0_history)
    si categorized.confidence > 0.85 :
        transaction.category = categorized.category
    sinon :
        transaction.category = "uncategorized"
        notify_merchant_for_review(transaction)
```

### 11.3 Daily allocation

```
daily_opex = Σ(monthly_recurring_opex) / 30
           + Σ(one_time_opex_on_day)
```

---

## 12. ARRONDIS & CONVERSIONS

### 12.1 Decimal → cents

```python
def to_cents(amount: Decimal) -> int:
    return int((amount * 100).quantize(Decimal("1"), rounding=ROUND_HALF_EVEN))
```

### 12.2 Cents → Decimal

```python
def from_cents(cents: int) -> Decimal:
    return Decimal(cents) / Decimal(100)
```

### 12.3 Percent calculation

```python
def compute_pct(numerator_cents: int, denominator_cents: int) -> Decimal:
    if denominator_cents == 0:
        return Decimal(0)
    return (Decimal(numerator_cents) / Decimal(denominator_cents)) * Decimal(100)
```

Arrondi à 2 décimales pour affichage :
```python
pct_display = compute_pct(num, denom).quantize(Decimal("0.01"), rounding=ROUND_HALF_EVEN)
```

### 12.4 Currency conversion

```python
def convert(amount: Decimal, from_cur: str, to_cur: str, on_date: date) -> Decimal:
    if from_cur == to_cur:
        return amount
    rate = fx_rates_service.get_rate(from_cur, to_cur, on_date)
    return (amount * rate).quantize(Decimal("0.0001"), rounding=ROUND_HALF_EVEN)
```

Rate résolution order :
1. Cached rate on_date (pgvector table `fx_rates_cache`)
2. Provider API (OpenExchangeRates / Fixer / ExchangeRate-API selon `FX_RATES_PROVIDER`)
3. Fallback : dernier taux connu + warning

---

## 13. TAX INCLUSIVE VS EXCLUSIVE

### 13.1 Règle

Shopify stocke le prix selon la config du merchant :
- **Tax-exclusive** (US default) : `line.price` n'inclut PAS la tax, `order.total_tax` est séparé
- **Tax-inclusive** (EU default) : `line.price` inclut la tax, `order.total_tax` est la part TVA extraite

### 13.2 Revenue normalisé

```
revenue_excl_tax = order.subtotal_price - order.total_tax   si tax_inclusive
                  = order.subtotal_price                     si tax_exclusive
```

**Toutes les formules de profit utilisent `revenue_excl_tax`.** La tax est un passage, pas une revenue.

---

## 14. TABLE RÉCAP — FORMULES PAR FEATURE

| Feature | Section(s) principale(s) |
|---------|--------------------------|
| #1 True Profit | 1.1, 1.2 |
| #2 App Cost Allocator | 4 |
| #3 Daily P&L | 2 |
| #4 Margin Alerts | 1.2 + comparison baseline |
| #5 Ad Spend 100% | 3 |
| #6 Return Cost | 6 |
| #7 OPEX Categorizer | 11 |
| #8 Tax Export | 13 + aggregation cross-period |
| #9 Cashflow Forecast | 7 |
| #10 Data Integrity | Voir `docs/DATA_INTEGRITY.md` |
| #11 Proactive Bug Alert | 4.4 |
| #12 LTV propre | 5 |
| #13 Multi-Fulfillment | 1.4 |
| #14 Post-Purchase Upsell | 10 |
| #15 Currency FX Loss | 9 |
| #35 Total Ad Spend 100% alerte | 3.4 |
| #40 Landed Cost | 8 |
| #41 Tariff Impact + Duty Tracker | 8.4 + 6 (duty unrecoverable) |

---

## 15. IMPLEMENTATION REFERENCE

### 15.1 Fichiers backend

```
backend/app/agent/analyzers/
├── true_profit.py            # sections 1, 2
├── app_cost.py               # section 4
├── ad_spend.py               # section 3
├── return_cost.py            # section 6
├── cashflow.py               # section 7
├── ltv_clean.py              # section 5
├── landed_cost.py            # section 8
├── tariff_impact.py          # section 8.4
├── fx_loss.py                # section 9
└── opex_categorizer.py       # section 11

backend/app/core/
└── currency.py               # section 12 (helpers Decimal ↔ cents, FX conversion)

backend/app/services/
└── fx_rates.py               # section 9 (provider + cache)
```

### 15.2 Tests obligatoires

Pour **chaque** formule :
- Test happy path avec valeurs round (ex: revenue=10000 ¢, cogs=4000 ¢ → gross=6000 ¢)
- Test avec rounding edge case (ex: 99.995 → doit donner 10000 ¢, pas 9999)
- Test multi-currency (ordre EUR, shop_currency USD, conversion au jour de l'order)
- Test refund partiel (2/5 items)
- Test Data Integrity failure (variance > 2% → ProfitError raised)

Fichiers tests :
```
backend/tests/test_profit/
├── test_true_profit.py
├── test_cogs_methods.py    # MAC, FIFO, Shopify native
├── test_ad_attribution.py
├── test_return_cost.py
├── test_cashflow.py
├── test_ltv.py
├── test_landed_cost.py
├── test_fx_loss.py
└── test_currency_helpers.py
```

---

## 16. CE QUI N'EST **PAS** SUPPORTÉ

Pour éviter la feature creep et rester chirurgical :

- **LIFO** (interdit IFRS)
- **Weighted average par batch** custom (seulement MAC global)
- **Inventory carrying cost** (stockage non-vendu) — Phase 2
- **Deferred revenue** (gift cards, subscriptions prepaid) — Phase 2
- **Intercompany transfer pricing** (multi-entity) — Phase 3 Enterprise
- **Fractional cents** (crypto, mills) — non supporté, tout en cents entiers

---

## 17. RÉFÉRENCES SOURCES

- **Decimal & rounding** : [Python docs decimal module](https://docs.python.org/3/library/decimal.html), IEEE 754-2019
- **Stripe fees on refunds** : [Stripe docs — refund fees policy](https://docs.stripe.com/refunds)
- **Drawback US (duty recovery)** : US CBP 19 CFR 191
- **HTS codes** : USITC DataWeb, Harmonized Tariff Schedule
- **IFRS FIFO/LIFO** : IFRS IAS 2 (Inventories)
- **Cohort LTV methodology** : Fader & Hardie, "Customer-Base Analysis"

---

**Dernière mise à jour :** 2026-04-24
**Status :** Source of truth figée pour M1. Modifications via RFC `docs/rfc/`.
**Reviewers obligatoires en cas de modification :** founder + 1 expert comptable externe.
