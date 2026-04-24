# FRAUD_RULES.md — Catalogue des règles de fraud scoring PilotProfit

> **Référence unique pour toutes les règles du Pre-Ship Score (feature #16).**
> **Architecture 4-tier inspirée de Marble + Jube + HeberTU, adaptée e-commerce Shopify.**
> **Aucune règle n'est implémentée dans le code sans être documentée ici.**

---

## PRINCIPE FONDATEUR

**Les faux positifs coûtent plus cher que la fraude.**

Stat NoFraud/Wyllo (7K brands G2) documentée : "too sensitive, cancelling genuine orders. Probably costing me more in lost orders than fraud orders." C'est notre anti-pattern #1. Chaque règle ajoutée doit être **testée** contre son impact sur le **False Positive Cost Tracker** (#37) avant d'être activée par défaut.

**Corollaire :** seuils calibrés par merchant (pas de one-size-fits-all), règles toutes override-ables, tous les blocages doivent être réviewables par le merchant (règle absolue #9 — aucun auto-reject permanent sans trace).

---

## ARCHITECTURE 4-TIER

```
                ┌─────────────────────────────────────┐
   Order ──▶   │  TIER 1 — BLOCKING RULES             │
                │  Déterministes, hard-coded           │──▶ Score = 100
                │  Auto-reject AVANT scoring           │    Reject immédiat
                │  (blacklist, velocity extrême,       │
                │   card testing confirmé)             │
                └─────────────────────────────────────┘
                             │ (si pas match)
                             ▼
                ┌─────────────────────────────────────┐
                │  TIER 2 — SCORING RULES              │
                │  Règles pondérées additives          │──▶ points_tier2 ∈ [0, 100]
                │  (freight forwarder +30, AMEX +15…)  │
                └─────────────────────────────────────┘
                             │
                             ▼
                ┌─────────────────────────────────────┐
                │  TIER 3 — ML MODEL (LightGBM)        │
                │  Features engineered, trained par    │──▶ ml_score ∈ [0, 1]
                │  merchant, drift-monitored           │
                └─────────────────────────────────────┘
                             │
                             ▼
                ┌─────────────────────────────────────┐
                │  AGRÉGATION                          │
                │  final_score = min(100,              │──▶ final_score ∈ [0, 100]
                │    0.5 × points_tier2 + 50 × ml)     │
                └─────────────────────────────────────┘
                             │
                             ▼
                ┌─────────────────────────────────────┐
                │  TIER 4 — HUMAN REVIEW               │
                │  Smart Hold (#17) si score > 75      │──▶ Merchant décide
                │  Cross-LLM review si order > $1000   │    Accept / Reject
                └─────────────────────────────────────┘
```

---

## TIER 1 — BLOCKING RULES (auto-reject, score = 100)

Ces règles renvoient `score = 100` et `recommendation = "reject"`. **Le merchant est notifié immédiatement** et voit pourquoi — mais la décision finale de cancel l'order sur Shopify reste **manuelle** (on n'appelle jamais `ordersCancel` en auto ; règle absolue #9 + pas d'écriture Shopify).

### T1.1 — Blacklist Match (certain)

**Déclencheur :** `blacklist.py` retourne un hit avec `confidence ≥ 0.95` sur au moins un identifiant.

**Identifiants matchés (hash tokenisé, salt `BLACKLIST_HASH_SALT`) :**
- `email_hash` (lowercased, normalized)
- `ip_hash` (IP v4/v6 as-is)
- `address_fingerprint_hash` (normalized street+city+zip+country)
- `phone_hash` (E.164 format)
- `card_fingerprint` (Stripe `charge.payment_method_details.card.fingerprint`)

**Règle :**
```
pour chaque identifier in [email, ip, address, phone, card_fingerprint] :
    hit = blacklist.lookup(identifier)
    si hit.confidence >= 0.95 AND hit.reported_by_merchants_count >= 3 :
        return BlockingRuleMatch(T1.1, identifier, hit)
```

**Justification :** 3 merchants différents confirment = pas un faux positif d'un merchant parano. Confidence < 0.95 → passe en Tier 2 (+40 points) au lieu de bloquer.

**Edge cases :**
- Merchant nouveau (<30j) ne contribue pas à la blacklist (évite le gaming) mais lit
- Dispute d'un customer qui conteste le blacklisting → process decrit dans `docs/BLACKLIST_PROCESS.md` (à venir)

---

### T1.2 — Velocity Extrême (carte)

**Déclencheur :** même `card_fingerprint` utilisé sur ≥ 10 orders en 1h cross-merchants.

**Règle :**
```
card_fp = order.payment_method.card_fingerprint
usage_last_1h = blacklist.card_usage_velocity(card_fp, window_minutes=60)
si usage_last_1h >= 10 :
    return BlockingRuleMatch(T1.2, card_fp, usage_last_1h)
```

**Justification :** Aucun customer légitime n'utilise sa carte 10× en 1h sur 10 merchants différents. Pattern typique de card testing automatisé.

**Edge cases :**
- Dropshipping cross-stores d'un même founder (rare) → whitelisting manuel
- Corporate card multi-commandes → à évaluer sur période plus longue (24h)

---

### T1.3 — Velocity Extrême (IP)

**Déclencheur :** même `ip` sur ≥ 15 orders en 15 min cross-merchants.

**Règle :**
```
ip_usage_last_15m = blacklist.ip_usage_velocity(order.client_ip, window_minutes=15)
si ip_usage_last_15m >= 15 :
    return BlockingRuleMatch(T1.3, order.client_ip, ip_usage_last_15m)
```

**Justification :** 15 orders en 15 min depuis la même IP = botnet ou attaque coordonnée.

**Edge cases :**
- IP de gateway d'entreprise (NAT) → exemption pour IPs marquées `corporate_nat` via WHOIS
- VPN partagé → pas d'exemption (les fraudeurs utilisent aussi des VPN)

---

### T1.4 — Card Testing Pattern Confirmé

**Déclencheur :** attaque de card testing active détectée par `card_testing.py` (feature #34).

**Règle :**
```
active_attack = card_testing.active_attack_detected(
    merchant_id=merchant_id,
    window_minutes=15,
)
si active_attack AND order.matches_attack_pattern(active_attack) :
    return BlockingRuleMatch(T1.4, active_attack.id, active_attack.pattern)
```

**Matches attack pattern** : IP dans le range attaquant, OU card fingerprint dans le burst détecté, OU cart value dans la fourchette du test.

**Justification :** Pendant une attaque active, tout order qui matche les signatures est bloqué même s'il aurait été OK hors attaque. Après fin de l'attaque (15 min sans nouveau signal), règle désactivée.

**Edge cases :**
- Customer légitime pendant une attaque → message UI "Order temporarily held — we detected suspicious activity. Please contact us if this is your order."
- Cross-LLM review de l'attaque elle-même valide l'ampleur avant de tout bloquer

---

### T1.5 — Test Mode Mismatch

**Déclencheur :** order avec `test: true` (Shopify test order) en production.

**Règle :**
```
si order.test is True AND merchant.env == "production" :
    return BlockingRuleMatch(T1.5, "test_order_in_prod", None)
```

**Justification :** Un test order en prod est soit un test oublié, soit une tentative d'exploit. Bloquer et notifier.

---

## TIER 2 — SCORING RULES (points pondérés)

Chaque règle match renvoie `{rule: name, points: int, reason: str}`. Les points s'additionnent, cappés à 100.

**Format d'une règle :**
```python
@dataclass
class ScoringRule:
    name: str
    points: int            # contribution au score
    category: str          # address | card | customer | ip | timing | behavioral
    min_confidence: float  # 0.0-1.0 — si match flou
    merchant_overridable: bool = True
```

---

### T2.1 — ADDRESS RULES

#### T2.1.1 — Freight Forwarder Address (+30)

**Détection :** `shipping_address.fingerprint` match dans la DB `freight_forwarders` (10K+ adresses connues : MyUS, Shipito, Stackry, Planet Express, etc.).

**Règle :**
```
fingerprint = normalize_address(order.shipping_address)
forwarder_match = db.freight_forwarders.lookup(fingerprint)
si forwarder_match.confidence >= 0.85 :
    return ScoringRuleMatch("T2.1.1", 30, "freight_forwarder", forwarder_match.name)
```

**Source validation terrain :** Thread $4200 (125 upvotes) : "On n'envoie plus aux commissionnaires de transport."

**Edge case :** `0.60 < confidence < 0.85` → +10 points (uncertain match).

#### T2.1.2 — Billing/Shipping Geographic Mismatch (+15)

**Détection :** `billing_address.country ≠ shipping_address.country` ET distance > 500 km.

**Règle :**
```
si order.billing.country != order.shipping.country :
    points = 15
si geo_distance(billing, shipping) > 500_km :
    points += 5
```

**Edge case :** Gifts (le customer peut mettre une note "gift for X") → -10 points si `order.note` contient `gift|cadeau|present`.

#### T2.1.3 — High-Risk Country (+10 à +25)

**Détection :** shipping ou billing dans liste pays haut-risque (configurable par merchant).

**Liste par défaut (à date 2026-04, révisée trimestriellement) :** basée sur Merchant Risk Council + Mastercard Fraud Index. Non hardcodée — stockée dans `fraud_risk_countries` DB table, mise à jour via cron.

**Règle :**
```
country_risk = db.fraud_risk_countries.lookup(order.shipping.country)
si country_risk.tier == "high" : points = 25
si country_risk.tier == "medium" : points = 10
```

**Edge case :** Merchant qui vend intentionnellement dans ces pays → règle désactivable dans settings.

#### T2.1.4 — Address Not Found / PO Box Pattern (+10)

**Détection :** address validator (SmartyStreets/Loqate) retourne `not_found` OU pattern PO Box pour un order contenant items non-dématérialisables > 100$.

**Règle :**
```
validation = address_validator.verify(order.shipping)
si validation.status == "not_found" : points = 10
si validation.is_po_box AND order.has_physical_items AND order.value_cents > 10000 : points = 10
```

---

### T2.2 — CARD / PAYMENT RULES

#### T2.2.1 — AMEX Card (+15, +25 si order > 500$)

**Détection :** `order.payment_method.card.brand == "amex"`.

**Règle :**
```
si order.card.brand == "amex" :
    points = 15
    si order.value_cents > 50000 : points = 25
```

**Source validation terrain :** Thread 53 upvotes : "On ne prend plus AMEX. J'ai perdu tous les bénéfices du mois à cause d'un seul chargeback AMEX." + Processor Health data cross-merchants.

**Edge case :** Merchant B2B où AMEX est dominant (ex: bureaux d'études) → règle désactivable.

#### T2.2.2 — AVS Mismatch (+20)

**Détection :** Stripe renvoie `address_line1_check: fail` OU `address_zip_check: fail`.

**Règle :**
```
avs_line1 = stripe_charge.payment_method_details.card.checks.address_line1_check
avs_zip = stripe_charge.payment_method_details.card.checks.address_zip_check

si avs_line1 == "fail" : points += 15
si avs_zip == "fail" : points += 10
```

**Edge case :** International orders où AVS n'est pas supporté → `avs_line1: unavailable` → pas de pénalité.

#### T2.2.3 — CVV Mismatch (+25)

**Détection :** `cvc_check: fail`.

**Règle :**
```
si stripe_charge.payment_method_details.card.checks.cvc_check == "fail" :
    points += 25
```

**Justification :** CVV fail = carte compromise très probable. +25 est agressif intentionnellement.

#### T2.2.4 — 3DS Not Used on High-Value Order (+10)

**Détection :** order.value > 500$ ET pas de 3DS activé sur le paiement.

**Règle :**
```
si order.value_cents > 50000 :
    three_d_secure = stripe_charge.payment_method_details.card.three_d_secure
    si three_d_secure is None OR three_d_secure.result != "authenticated" :
        points += 10
```

**Liaison feature #33 :** déclenche aussi la suggestion "Enable 3DS" dans l'UI.

#### T2.2.5 — Prepaid / Gift Card (+15)

**Détection :** `stripe_charge.payment_method_details.card.funding == "prepaid"`.

**Justification :** Cartes prépayées = intraçables, impossible d'obtenir une liability shift chargeback. Pattern fraudeur fréquent.

---

### T2.3 — CUSTOMER RULES

#### T2.3.1 — First-Time Customer Large Cart (+10)

**Détection :** `customer.orders_count == 0` ET `order.value > P90 des orders du merchant sur 90j`.

**Règle :**
```
customer_order_count = shopify.customer.orders_count(customer_id)
p90 = stats.percentile(merchant.orders_last_90d.map(o => o.value), 90)

si customer_order_count == 0 AND order.value_cents > p90 :
    points = 10
```

**Edge case :** Merchant luxe avec AOV très élevé et clientèle majoritairement first-time → seuil configurable.

#### T2.3.2 — Email Domain Récent (+10)

**Détection :** domaine de l'email créé depuis < 7 jours.

**Règle :**
```
domain = extract_domain(order.email)
creation_date = whois_service.get_creation_date(domain)
si creation_date is not None AND (now() - creation_date).days < 7 :
    points = 10
```

**Edge cases :**
- Nouveau TLD (.shop, .store) qui a boom récemment → exemption pour top 1000 TLDs
- Domaines custom entreprise → WHOIS privacy peut masquer, fallback sur heuristiques (MX records, SPF)

#### T2.3.3 — Disposable Email Provider (+20)

**Détection :** email chez Mailinator, 10minutemail, Guerrilla Mail, etc. (liste de 3K+ domains).

**Règle :**
```
domain = extract_domain(order.email)
si domain in disposable_email_domains_list :
    points = 20
```

Liste maintenue dans `fraud_disposable_emails` DB table (syncée depuis projets open-source comme `disposable-email-domains/disposable-email-domains`).

#### T2.3.4 — Phone Number Invalid or VoIP (+10)

**Détection :** Twilio Lookup API ou équivalent → `line_type: voip` OU `valid: false`.

**Règle :**
```
lookup = phone_validator.check(order.phone)
si lookup.valid == False : points = 10
si lookup.line_type == "voip" : points = 5
```

**Edge case :** Google Voice et similaires (VoIP grand public) → +5 seulement.

---

### T2.4 — IP / NETWORK RULES

#### T2.4.1 — IP Country Mismatch Billing (+15)

**Détection :** geolocation(IP) ≠ billing_address.country.

**Règle :**
```
ip_country = geoip.lookup(order.client_ip).country
si ip_country != order.billing.country :
    points = 15
```

**Edge case :** VPN/Proxy flag séparé → voir T2.4.2.

#### T2.4.2 — VPN / Proxy / Tor Detected (+15)

**Détection :** IP2Proxy, IPQualityScore, ou MaxMind `traits.is_anonymous_proxy`.

**Règle :**
```
ip_info = geoip.advanced_lookup(order.client_ip)
si ip_info.is_vpn : points += 10
si ip_info.is_tor : points += 30   # Tor = très rare pour shopping légitime
si ip_info.is_public_proxy : points += 15
si ip_info.is_datacenter : points += 10
```

**Edge case :** Corporate VPN (ex: employés en remote via VPN entreprise) → détection par ASN owner (si "Corporate IT" type), exemption -10.

#### T2.4.3 — IP Reputation Bad (+20)

**Détection :** `ip_reputation.score < 0.3` selon AbuseIPDB ou Spamhaus.

**Règle :**
```
reputation = ip_reputation_service.score(order.client_ip)
si reputation.score < 0.3 AND reputation.reports_last_30d > 5 :
    points = 20
```

---

### T2.5 — CROSS-ORDER PATTERNS

#### T2.5.1 — Same IP, Multiple Orders Recent (+20)

**Détection :** même IP sur ≥ 3 orders du merchant dans les 60 dernières min (non-Blocking mais suspect).

**Règle :**
```
same_ip_count = merchant.orders.filter(
    client_ip=order.client_ip,
    created_at > now() - 60min,
).count()
si same_ip_count >= 3 : points = 20
si same_ip_count >= 6 : points = 40   # approche Tier 1
```

**Edge case :** Même household (parents qui commandent depuis la même IP pour plusieurs enfants) → pattern reconnu via Mem0 après quelques orders validés, règle adaptée.

#### T2.5.2 — Same Address, Different Payment Methods (+25)

**Détection :** même `shipping_address.fingerprint` avec ≥ 3 cartes différentes sur 7 jours.

**Règle :**
```
cards_at_address = merchant.orders.filter(
    shipping_fingerprint=order.shipping.fingerprint,
    created_at > now() - 7d,
).distinct("payment_method.card_fingerprint").count()
si cards_at_address >= 3 : points = 25
```

**Justification :** Pattern classique de package reshipment fraud.

#### T2.5.3 — Order Number Velocity (+15)

**Détection :** même customer enchaîne 3+ orders en < 5 min.

**Règle :**
```
customer_recent_orders = shopify.customer.orders(customer_id, since=now() - 5min).count()
si customer_recent_orders >= 3 : points = 15
```

**Edge case :** Cart abandonment + retry légitime → si les 3 orders ont le même content, regrouper, pas de pénalité.

---

### T2.6 — TIMING RULES

#### T2.6.1 — Odd Hour Order (+5)

**Détection :** order créé entre 2h-5h du matin dans la timezone du billing country.

**Règle :**
```
local_hour = convert_to_local(order.created_at, order.billing.country).hour
si 2 <= local_hour <= 5 : points = 5
```

**Faible poids :** Les insomniaques et shift workers existent. C'est un signal faible, cumulatif.

#### T2.6.2 — Rush Before Closing (+10)

**Détection :** order placé dans les 5 min avant la fin d'une promo/sale + cart à la limite haute.

**Règle :**
```
active_promo = promo_service.get_active(merchant_id, at=order.created_at)
si active_promo AND (active_promo.end - order.created_at).minutes < 5 :
    si order.value > 0.8 × merchant.aov_90d : points = 10
```

**Justification :** Pattern de "promo abuse" automatisé.

---

### T2.7 — BEHAVIORAL RULES

#### T2.7.1 — Unusual Cart Composition (+10)

**Détection :** cart contient mix de items avec margin très différente (ex: 1 item haute-margin + 1 item bas-margin à prix plancher), souvent pattern de "revente de la partie premium".

**Règle :**
```
items_margins = [compute_margin(item) for item in order.line_items]
max_margin = max(items_margins)
min_margin = min(items_margins)
si len(order.line_items) >= 2 AND (max_margin - min_margin) > 0.5 :
    points = 10
```

**Edge case :** Merchant bundle deals légitimes → pattern reconnu par Mem0, règle atténuée après 30+ orders similaires validés.

#### T2.7.2 — Guest Checkout + High Value (+10)

**Détection :** `order.customer.is_guest == True` ET `order.value > 1.5 × merchant.aov_90d`.

**Règle :**
```
si order.customer is None OR order.customer.is_guest :
    si order.value_cents > 1.5 * merchant.aov_90d_cents : points = 10
```

---

## TIER 3 — ML MODEL (LightGBM)

Détails complets dans `docs/ML_MODELS.md`. Synthèse ici pour la traçabilité du scoring.

### T3 — Score ML

**Model :** LightGBM binary classifier (target : `is_chargeback_confirmed_90d` OU `is_fraud_manually_marked`).

**Output :** `ml_score ∈ [0, 1]`.

**Features engineered (top 20) :**

1. `order.value_cents_normalized` (par-merchant z-score)
2. `order.items_count`
3. `customer.orders_count`
4. `customer.total_spent_cents`
5. `customer.days_since_first_order`
6. `customer.days_since_last_order`
7. `customer.chargebacks_lifetime_count`
8. `email_domain_age_days`
9. `ip_country_matches_billing` (bool)
10. `shipping_matches_billing` (bool)
11. `avs_line1_check_match` (bool)
12. `avs_zip_check_match` (bool)
13. `cvc_check_match` (bool)
14. `three_d_secure_used` (bool)
15. `card_funding_type` (credit / debit / prepaid — ordinal)
16. `card_brand` (visa / mastercard / amex / discover — categorical)
17. `hour_of_day_local`
18. `day_of_week_local`
19. `is_weekend` (bool)
20. `cart_margin_variance`

**Training :**
- Modèle **par merchant** après 500+ orders historiques + 30+ chargebacks
- Avant calibration : modèle global (shared) fallback
- Retraining hebdomadaire via Celery `fraud_tasks::retrain_model`
- Evaluation : AUC-ROC, calibration curve, drift monitoring (PSI)

**Override :**
- ML score < 0.2 ET points_tier2 < 10 → passe direct sans review
- ML score > 0.8 ET points_tier2 > 40 → flag "ML + rules concur" (haute confidence)

---

## AGRÉGATION DU SCORE

### Formule finale

```python
def compute_final_score(tier2_points: int, ml_score: float, merchant_weights: MerchantWeights) -> int:
    """
    tier2_points ∈ [0, 100]
    ml_score ∈ [0, 1]
    """
    w2 = merchant_weights.tier2_weight   # default 0.5
    w3 = merchant_weights.ml_weight      # default 0.5

    score = (w2 * tier2_points) + (w3 * ml_score * 100)
    return min(100, int(round(score)))
```

### Calibration par merchant

Au premier setup : `w2 = 1.0, w3 = 0.0` (ML pas calibré).

Après 500 orders + 30 chargebacks : recalibration basée sur F1 optimal, typiquement converge vers `w2 = 0.4-0.6, w3 = 0.4-0.6`.

Stocké dans table `merchant_fraud_config` avec historique des changements.

---

## RISK LEVEL MAPPING

| Score | Risk Level | Action recommandée |
|-------|------------|-------------------|
| 0-39 | low | Pass sans friction |
| 40-59 | medium | Notification merchant (non-bloquante) |
| 60-74 | elevated | Notification + suggestion 3DS / verif |
| 75-89 | high | Smart Hold → merchant review |
| 90-100 | critical | Smart Hold + cross-LLM review + préconisation reject |

**Seuil de hold configurable par merchant** (défaut : 75). Cross-LLM review triggered automatiquement si `order.value > 1000$` OU `risk_level == "critical"`.

---

## PATTERNS SPÉCIAUX (Analyzers dédiés)

Ces patterns ont leur propre analyzer, mais **feedent le Pre-Ship Score** via Tier 2 ou Tier 1.

### P1 — Card Testing (feature #34)

Détaillé ci-dessus en T1.4 pour l'attaque active. Le scanner `card_testing.py` maintient un **état global** de détection d'attaques :

**Signaux d'attaque :**
```python
{
    "failed_payments_last_15min": > 30,
    "success_rate_last_15min_pct": < 20,
    "distinct_cards_tested": > 20,
    "common_ip_ranges_concentration": > 0.6,  # % d'orders depuis un seul /24
    "cart_value_range": cohérence (ex: tous < $10 = testing)
}
```

**Si 3+ signaux → attaque active** → T1.4 activé pour tout order matchant.

### P2 — Friendly Fraud (feature #24)

L'analyzer `friendly_fraud.py` scanne l'historique customer :

**Pattern :**
```python
customer_chargeback_history = {
    "delivered_orders_count": n,
    "support_contacts_before_chargeback_count": 0,
    "time_delivery_to_chargeback_avg_days": > 14,
    "chargeback_reason_codes": majority in ["product_not_received", "product_not_as_described"],
}
```

**Si le customer a ce pattern ≥ 2 fois** → flag Tier 2 (+25) sur nouveaux orders de ce customer.

### P3 — Return Abuse (feature #39)

Pattern wardrobing / serial returner. Scanner `return_abuse.py`.

**Signaux :**
```python
{
    "orders_6m": > 3,
    "return_rate_pct": > 60,
    "returns_sizes_match_events": True,   # Aug = grande robe, Sep = robe retournée
    "returns_with_damage_count": > 0,
}
```

**Si détecté** → flag Tier 2 (+15) sur nouveaux orders du customer, alerte merchant.

---

## VAMP & THRESHOLDS NETWORK (feature #22)

### Thresholds (2026-04, à vérifier trimestriellement)

| Network | Monitoring threshold | Fines threshold |
|---------|---------------------|-----------------|
| **Visa VAMP** | 0.65 % dispute rate | 0.9 % → fines 25-100K $/mois |
| **Mastercard ECM** | 0.75 % dispute rate | 1.0 % → fines progressives |
| **AMEX** | 1.0 % chargeback rate | 1.5 % → compte review |
| **Discover** | 1.0 % dispute rate | 1.5 % |

### Calcul du ratio

```
visa_ratio_30d = (visa_disputes_30d / visa_completed_transactions_30d) × 100
```

**Important :**
- Disputes = formally filed, pas "inquiries"
- Completed transactions = capturés, pas juste autorisés
- Rolling 30j, pas calendar month

### Stripe dispute rate vs Visa dispute ratio

Stripe calcule **leur** dispute rate légèrement différemment (inclut certaines inquiries pre-dispute). **On track les deux**, on alerte sur le plus strict.

### Alerting logic

```python
for network in ["visa", "mastercard", "amex", "discover"]:
    current_ratio = compute_ratio(merchant_id, network, window_days=30)
    projected_30d_ratio = linear_projection(
        history=last_60d_ratios,
        horizon_days=30,
    )

    if current_ratio >= 0.8 * network.fines_threshold:
        alert(severity="critical", reason=f"{network} approaching fines threshold")
    elif current_ratio >= network.monitoring_threshold:
        alert(severity="warning", reason=f"{network} in monitoring program")
    elif projected_30d_ratio >= 0.8 * network.fines_threshold:
        alert(severity="warning", reason=f"{network} projected to hit fines threshold in 30d")
```

**Cross-LLM review** valide les alertes critiques avant de stresser le merchant (évite les faux positifs sur petits volumes).

---

## MERCHANT OVERRIDE

Toutes les règles Tier 2 sont **overridables** par merchant :
- Désactivation globale (`disabled`)
- Modification du point weight (ex: `amex_card` de +15 à +5)
- Allowlist spécifique (ex: "toujours accepter mon top client X même si pattern match")
- Blocklist spécifique (ex: "toujours bloquer commandes depuis cette ville qui m'a fraud")

**Stockage :** table `merchant_fraud_config` avec JSON field `rule_overrides`.

**UI :** `/dashboard/fraud/rules` — le merchant voit toutes les règles actives, peut toggle/ajuster. Chaque modification loggée avec raison optionnelle.

**Ne sont PAS overridables (hardcoded) :**
- Tier 1 rules (blacklist, velocity extrême, card testing, test mode) — sécurité baseline
- Compute du score final (formule agrégation)

---

## FEEDBACK LOOP (retraining et calibration)

Chaque action merchant sur un order scoré alimente le feedback :

| Action merchant | Feedback tag |
|-----------------|--------------|
| Release hold → order livré, pas de chargeback en 45j | `false_positive_confirmed` |
| Release hold → chargeback reçu plus tard | `correct_block_released_by_mistake` |
| Reject hold → customer confirme fraud ou dispute perdue | `confirmed_fraud` |
| Reject hold → customer conteste et c'est validé | `false_positive_penalty` |

Ces tags :
1. **Re-entraînent le ML model** (hebdo)
2. **Ajustent le seuil de hold** par merchant (si FP rate > target, raise threshold)
3. **Updatent Mem0** avec les préférences merchant (ex: "ce merchant tolère plus de risque sur les AMEX car B2B")

---

## TABLE RÉCAPITULATIVE DES POINTS

| Règle | Tier | Points | Catégorie | Feature parent |
|-------|------|--------|-----------|----------------|
| Blacklist match high confidence | T1.1 | 100 (block) | — | #23 |
| Velocity extrême carte | T1.2 | 100 (block) | — | — |
| Velocity extrême IP | T1.3 | 100 (block) | — | — |
| Card testing attack active | T1.4 | 100 (block) | — | #34 |
| Test mode mismatch | T1.5 | 100 (block) | — | — |
| Freight forwarder address | T2.1.1 | 30 | address | #31 |
| Billing/shipping geo mismatch | T2.1.2 | 15-20 | address | — |
| High-risk country | T2.1.3 | 10-25 | address | — |
| Address not found / PO Box | T2.1.4 | 10 | address | — |
| AMEX card | T2.2.1 | 15-25 | card | #30 |
| AVS mismatch | T2.2.2 | 10-25 | card | — |
| CVV mismatch | T2.2.3 | 25 | card | — |
| 3DS not used high-value | T2.2.4 | 10 | card | #33 |
| Prepaid / gift card | T2.2.5 | 15 | card | — |
| First-time customer large cart | T2.3.1 | 10 | customer | — |
| Email domain récent | T2.3.2 | 10 | customer | — |
| Disposable email | T2.3.3 | 20 | customer | — |
| Phone invalid/VoIP | T2.3.4 | 5-10 | customer | — |
| IP country mismatch billing | T2.4.1 | 15 | ip | — |
| VPN/Proxy/Tor | T2.4.2 | 10-30 | ip | — |
| IP reputation bad | T2.4.3 | 20 | ip | — |
| Same IP multiple orders | T2.5.1 | 20-40 | cross-order | — |
| Same address different cards | T2.5.2 | 25 | cross-order | — |
| Order velocity customer | T2.5.3 | 15 | cross-order | — |
| Odd hour | T2.6.1 | 5 | timing | — |
| Rush before closing | T2.6.2 | 10 | timing | — |
| Unusual cart composition | T2.7.1 | 10 | behavioral | — |
| Guest checkout high value | T2.7.2 | 10 | behavioral | — |
| Friendly fraud pattern (customer) | P2 | 25 | pattern | #24 |
| Return abuse pattern (customer) | P3 | 15 | pattern | #39 |

**Maximum théorique Tier 2 si toutes les règles matchent :** ~400 points → cappé à 100.

---

## IMPLEMENTATION REFERENCE

### 15.1 Fichiers backend

```
backend/app/agent/analyzers/
├── fraud_scorer.py               # Orchestration des 4 tiers
├── card_testing.py               # P1 + T1.4
├── friendly_fraud.py             # P2
├── return_abuse.py               # P3
├── ratio_monitor.py              # Feature #22 VAMP

backend/app/agent/ml/
├── fraud_model.py                # T3 — LightGBM
├── feature_engineering.py        # T3 features (liste ci-dessus)
├── training_pipeline.py          # Retraining hebdo

backend/app/services/
├── blacklist.py                  # T1.1, T1.2, T1.3 lookups
├── geoip.py                      # T2.4.x (MaxMind / IPQualityScore)
├── address_validator.py          # T2.1.1, T2.1.4 (SmartyStreets / Loqate)
├── email_validator.py            # T2.3.2, T2.3.3 (WHOIS + disposable list)
└── phone_validator.py            # T2.3.4 (Twilio Lookup)
```

### 15.2 Tests obligatoires

Pour **chaque règle Tier 1 et Tier 2** :
- Test match happy path (règle s'active sur input de référence)
- Test non-match (règle ne s'active pas sur input safe)
- Test edge case (borderline → uncertain)
- Test merchant override (règle désactivée → pas de points)

Pour le ML model :
- Test calibration curve
- Test drift detection (PSI > 0.2 → alerte retraining urgent)
- Test fallback vers global model si merchant model pas prêt

```
backend/tests/test_fraud/
├── test_tier1_blocking.py
├── test_tier2_address.py
├── test_tier2_card.py
├── test_tier2_customer.py
├── test_tier2_ip.py
├── test_tier2_cross_order.py
├── test_tier2_timing.py
├── test_tier2_behavioral.py
├── test_aggregation.py           # formule finale
├── test_ratio_monitor.py         # VAMP thresholds
├── test_card_testing.py
├── test_friendly_fraud.py
├── test_ml_model.py
└── test_merchant_override.py
```

---

## SOURCES & REFERENCES

- **Marble** (`checkmarble/marble`) — rules engine open source (AGPLv3) — inspiration Tier 1 & Tier 2 architecture
- **Jube** (`jube-home/aml-fraud-transaction-monitoring`) — velocity checks, aggregation patterns
- **HeberTU/fraud-detection-system** — architecture 4-tier (blocking / scoring / ML / investigators)
- **Mastercard Chargeback Guide 2025** — thresholds et reason codes
- **Visa VAMP Program** — [Visa Dispute Monitoring Program](https://www.visa.com/merchantmonitoring)
- **Stripe Radar documentation** — rules patterns pour e-commerce
- **Reddit r/ecommerce** — thread 230 upvotes (friendly fraud), thread $4200 (freight forwarders), thread 53 upvotes (AMEX)

---

## DÉTECTION DE NOUVELLES RÈGLES (roadmap Phase 2)

Patterns identifiés lors du scraping mais non implémentés M1 :
- **Device fingerprinting** (FingerprintJS-like) — Phase 2
- **Behavioral biometrics** (typing cadence, mouse movement) — Phase 3
- **Graph analysis** (customer/address/card graphs pour détection rings) — Phase 2
- **Early fraud warnings Ethoca/Verifi** (TC40/SAFE alerts) — Phase 2, dépend accord commercial

---

**Dernière mise à jour :** 2026-04-24
**Status :** Règles figées pour M1 launch. Modifications via RFC `docs/rfc/`.
**Reviewers obligatoires en cas de modification :** founder + 1 expert fraud/risk externe + test False Positive Cost tracker pendant 14j avant rollout.
