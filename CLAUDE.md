# CLAUDE.md — PilotProfit

> **Ce fichier est chargé automatiquement à chaque session Claude Code.**
> **Lis-le EN ENTIER avant de toucher au code.**

---

## PROJET

PilotProfit — Agent IA santé financière complète pour merchants Shopify.
41 features, 4 modules : Profit, Anti-Fraude, Intelligence, Tarifs & Landed Cost.
App Shopify (OAuth, embedded) + dashboard standalone. Pricing : Free → Starter 49$ → Pro 129$ → Enterprise 249$.

Système multi-agents orchestré par un **Supervisor Agent** (LangGraph) qui route vers 5 agents spécialistes, chacun régi par sa **Constitution** (fichier markdown chargé en system prompt).

Repo GitHub : `pilotprofit`. Standalone, pas de monorepo.

**Infrastructure pre-launch :** Frontend sur `pilotprofit.vercel.app`, API sur `pilotprofit-api.up.railway.app`. Domaine `pilotprofit.app` sera acheté après le lancement public.

---

## STACK TECHNIQUE

### Backend
- **Python 3.12+** — FastAPI (async, Pydantic v2)
- **Celery** + **Redis** — background jobs (profit calc, ad sync, fraud scoring, evidence building, reports)
- **LangGraph** — orchestration multi-agents (Supervisor Pattern)
- **Claude API** (Anthropic, Opus) — LLM primaire (generation, evidence drafting, alerts langage simple, advisory)
- **OpenAI GPT-5 + Google Gemini** — LLMs secondaires pour le **Cross-LLM Review** des décisions high-stakes
- **Mem0** — mémoire persistante agent (merchant patterns, fraud ML calibration, baseline profit, preferences)
- **scikit-learn / LightGBM** — ML fraud scoring (Tier 3 du rules engine)
- **httpx** — HTTP client async (Shopify, Stripe, Meta, Google, TikTok)
- **Deploy : Railway** (backend API + Celery worker + Celery beat, services séparés)

### Patterns d'architecture agent (inspirés des meilleurs patterns multi-agents 2026)

1. **Supervisor Pattern** — Un brain LangGraph orchestre 5 agents spécialistes. Route chaque tâche vers le bon spécialiste. Observe tout via `/admin/agents` (dashboard interne).
2. **Agent Constitutions** — Chaque agent a un fichier `.md` non-négociable chargé en system prompt à chaque invocation. Garantit la cohérence du comportement dans le temps. Pattern inspiré d'Ouroboros `BIBLE.md`, adapté SaaS production.
3. **Cross-LLM Review** — Décisions high-stakes (evidence chargeback > 500$, hold > 1000$, alerts critiques) : Claude génère → GPT-5 ou Gemini review → consensus ou escalade humaine. Pattern inspiré d'Ouroboros multi-model review.

### Frontend
- **Next.js 14** — App Router, TypeScript strict, SSR pour SEO landing page
- **Tailwind CSS** + **shadcn/ui** — design system
- **Shopify App Bridge** — embedded app dans le Shopify Admin
- **Shopify Polaris** — guidelines UI pour la cohérence Shopify
- **Recharts** — visualisations profit/fraud dashboards
- **Deploy : Vercel** (pre-launch : `pilotprofit.vercel.app`)

### Database
- **Supabase PostgreSQL** — tables métier + auth + billing + agent
- **RLS (Row Level Security)** — OBLIGATOIRE sur chaque table
- **pgvector** — stockage Mem0 (si self-hosted) + embeddings pour similarity search fraude
- **TimescaleDB extension** (optionnel phase 2) — time-series pour profit trends

### Services externes
- **Shopify Admin API** — GraphQL (version 2026-01)
- **Stripe** — Checkout, Customer Portal, Webhooks (billing) + **Stripe Disputes API** (chargebacks)
- **Meta Marketing API** — ad spend 100% (via `facebook-business` SDK)
- **Google Ads API** — ad spend 100% (via `google-ads-python`)
- **TikTok Business API** — ad spend 100%
- **Resend** — emails transactionnels (alerts, weekly reports, customer "polite threat")
- **Sentry** — error tracking
- **LangSmith** — LLM tracing + cost tracking par agent (critique pour le Supervisor dashboard)

---

## STRUCTURE DU PROJET

```
pilotprofit/                            # Racine du repo GitHub
├── CLAUDE.md                           # CE FICHIER
├── context.md                          # Vision business, features, personas, pricing
│
├── backend/
│   ├── app/
│   │   ├── main.py                     # FastAPI app, startup, middleware, CORS
│   │   ├── config.py                   # Settings Pydantic BaseSettings (env vars)
│   │   ├── dependencies.py             # Injection de dépendances (DI)
│   │   │
│   │   ├── api/
│   │   │   ├── routes/
│   │   │   │   ├── auth.py             # Shopify OAuth install/callback + session
│   │   │   │   ├── profit.py           # True profit, P&L, LTV, margins, FX loss
│   │   │   │   ├── fraud.py            # Pre-Ship Score, holds, card testing, FP tracker
│   │   │   │   ├── chargebacks.py      # Disputes list, evidence, approval workflow
│   │   │   │   ├── ads.py              # Ad spend aggregation, integration health
│   │   │   │   ├── tariffs.py          # Landed cost, tariff impact, duty tracker
│   │   │   │   ├── tax.py              # Tax export (QuickBooks, Xero, CSV)
│   │   │   │   ├── integrity.py        # Data integrity checks, reconciliation alerts
│   │   │   │   ├── integrations.py     # OAuth connect/disconnect Meta/Google/TikTok/etc.
│   │   │   │   ├── billing.py          # Stripe checkout, portal
│   │   │   │   ├── notifications.py    # Notification preferences, list
│   │   │   │   ├── reports.py          # Weekly reports, export
│   │   │   │   ├── feedback.py         # Feedback loop accept/reject
│   │   │   │   ├── admin_agents.py     # Supervisor dashboard : agents status, budget, tasks
│   │   │   │   ├── webhooks_shopify.py # Shopify webhooks receiver
│   │   │   │   ├── webhooks_stripe.py  # Stripe webhooks (billing + disputes)
│   │   │   │   ├── webhooks_ads.py     # Meta/Google/TikTok webhooks
│   │   │   │   └── health.py           # Healthcheck endpoint
│   │   │   └── middleware/
│   │   │       ├── auth.py             # JWT validation Supabase
│   │   │       ├── rate_limit.py       # Rate limiting Redis
│   │   │       └── hmac.py             # HMAC validation webhooks
│   │   │
│   │   ├── agent/
│   │   │   ├── supervisor.py           # Supervisor LangGraph — brain général, routing
│   │   │   ├── state.py                # AgentState dataclass (shared across agents)
│   │   │   ├── memory.py               # MerchantMemory — Mem0 client wrapper
│   │   │   ├── learner.py              # Feedback loop — retraining ML + Mem0 update
│   │   │   ├── cross_llm_review.py     # Cross-LLM review pour décisions high-stakes
│   │   │   ├── constitutions_loader.py # Charge les CONSTITUTION.md en system prompt
│   │   │   │
│   │   │   ├── specialists/            # 5 agents spécialistes
│   │   │   │   ├── profit_analyst.py   # Agent Profit Analyst
│   │   │   │   ├── fraud_investigator.py  # Agent Fraud Investigator
│   │   │   │   ├── chargeback_specialist.py # Agent Chargeback Specialist
│   │   │   │   ├── data_integrity.py   # Agent Data Integrity
│   │   │   │   └── ads_sync.py         # Agent Ads Sync
│   │   │   │
│   │   │   ├── constitutions/          # CONSTITUTION.md des agents (soul files)
│   │   │   │   ├── profit_analyst.md
│   │   │   │   ├── fraud_investigator.md
│   │   │   │   ├── chargeback_specialist.md
│   │   │   │   ├── data_integrity.md
│   │   │   │   └── ads_sync.md
│   │   │   │
│   │   │   ├── detectors/
│   │   │   │   ├── shopify_events.py   # Shopify webhooks → trigger Supervisor
│   │   │   │   ├── stripe_events.py    # Stripe events (disputes, refunds)
│   │   │   │   ├── ads_sync_cron.py    # Daily cron ad spend sync
│   │   │   │   ├── cron_pnl.py         # Nightly P&L calculation
│   │   │   │   └── integrity_monitor.py # Cross-source reconciliation cron
│   │   │   │
│   │   │   ├── analyzers/              # Fonctions pures appelées par les specialists
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py             # BaseAnalyzer ABC
│   │   │   │   ├── true_profit.py      # True profit per order/product/channel
│   │   │   │   ├── ltv_clean.py        # LTV without bots / non-buyers
│   │   │   │   ├── cashflow.py         # 30/60/90d cashflow forecast
│   │   │   │   ├── app_cost.py         # Shopify apps cost allocator
│   │   │   │   ├── ad_spend.py         # 100% ad spend aggregator
│   │   │   │   ├── margin_monitor.py   # Margin threshold alerts
│   │   │   │   ├── return_cost.py      # Real cost of returns
│   │   │   │   ├── landed_cost.py      # COGS + shipping + duties + tariffs
│   │   │   │   ├── tariff_impact.py    # Simulate tariff changes impact
│   │   │   │   ├── fx_loss.py          # Currency FX loss tracker (multi-devises)
│   │   │   │   ├── fraud_scorer.py     # Pre-Ship Score (rules + ML)
│   │   │   │   ├── card_testing.py     # Card testing pattern detector
│   │   │   │   ├── friendly_fraud.py   # Friendly fraud pattern detection
│   │   │   │   ├── return_abuse.py     # Wardrobing / serial returners
│   │   │   │   ├── ratio_monitor.py    # Chargeback ratio vs Visa/MC thresholds
│   │   │   │   ├── processor_health.py # Payment processor chargeback rates
│   │   │   │   ├── data_integrity.py   # Shopify vs Stripe vs DB reconciliation
│   │   │   │   └── opex_categorizer.py # Auto-categorize operating expenses
│   │   │   │
│   │   │   ├── actors/                 # Side effects (write actions)
│   │   │   │   ├── notification.py     # Push + email + in-app
│   │   │   │   ├── smart_hold.py       # Auto-hold high-risk orders
│   │   │   │   ├── evidence_builder.py # Chargeback evidence PDF (human approval required)
│   │   │   │   ├── polite_threat.py    # Auto-email to chargebacker customer
│   │   │   │   ├── tax_export.py       # QuickBooks/Xero/CSV export
│   │   │   │   ├── report_generator.py # Weekly report HTML + PDF
│   │   │   │   └── advisory.py         # Claude-powered strategic advisory
│   │   │   │
│   │   │   └── ml/
│   │   │       ├── fraud_model.py      # LightGBM classifier
│   │   │       ├── feature_engineering.py # Feature extraction for scoring
│   │   │       └── training_pipeline.py # Periodic retraining with feedback
│   │   │
│   │   ├── services/
│   │   │   ├── shopify.py              # Shopify GraphQL client + rate limit + retry
│   │   │   ├── stripe_billing.py       # Stripe subscription + customer portal
│   │   │   ├── stripe_disputes.py      # Stripe Disputes API client
│   │   │   ├── meta_ads.py             # Meta Marketing API client
│   │   │   ├── google_ads.py           # Google Ads API client
│   │   │   ├── tiktok_ads.py           # TikTok Business API client
│   │   │   ├── quickbooks.py           # QuickBooks export
│   │   │   ├── xero.py                 # Xero export
│   │   │   ├── tsi_recovery.py         # TSI/Rocket recovery integration
│   │   │   ├── openai_client.py        # OpenAI client pour cross-LLM review
│   │   │   ├── gemini_client.py        # Gemini client pour cross-LLM review
│   │   │   ├── fx_rates.py             # Multi-currency exchange rates (daily)
│   │   │   ├── supabase.py             # Supabase client + helpers
│   │   │   ├── email.py                # Resend email service
│   │   │   └── blacklist.py            # Cross-merchant tokenized blacklist
│   │   │
│   │   ├── models/
│   │   │   ├── order.py                # Order, OrderItem, OrderProfit
│   │   │   ├── merchant.py             # Merchant, Subscription
│   │   │   ├── fraud.py                # FraudScore, Hold, Blacklist, CardTestingEvent
│   │   │   ├── chargeback.py           # Chargeback, Evidence, EvidenceApproval
│   │   │   ├── ads.py                  # AdSpend, AdCampaign, AdAttribution
│   │   │   ├── profit.py               # ProfitCalculation, DailyPnL, LTV, FXLoss
│   │   │   ├── integration.py          # Integration, IntegrationHealth
│   │   │   ├── tariff.py               # LandedCost, DutyPaid, TariffRate
│   │   │   ├── agent_run.py            # AgentRun, AgentDecision, ReviewConsensus
│   │   │   └── schemas.py              # Pydantic request/response schemas
│   │   │
│   │   └── core/
│   │       ├── exceptions.py           # AppError hierarchy + ErrorCode enum
│   │       ├── security.py             # Fernet encrypt/decrypt, HMAC, JWT helpers
│   │       ├── currency.py             # Multi-currency conversion (daily rates)
│   │       └── logging.py              # structlog config
│   │
│   ├── tasks/
│   │   ├── celery_app.py               # Celery config + beat schedule
│   │   ├── profit_tasks.py             # Nightly P&L, LTV refresh, FX loss aggregation
│   │   ├── ads_tasks.py                # Daily ad spend sync
│   │   ├── fraud_tasks.py              # Real-time scoring, card testing detection, retraining
│   │   ├── chargeback_tasks.py         # Evidence building, deadline reminders
│   │   ├── integrity_tasks.py          # Hourly reconciliation checks
│   │   └── report_tasks.py             # Weekly report generation
│   │
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── test_profit/
│   │   ├── test_fraud/
│   │   ├── test_chargebacks/
│   │   ├── test_ads/
│   │   ├── test_integrity/
│   │   ├── test_supervisor/            # Tests du routing Supervisor + Constitutions
│   │   ├── test_cross_llm/             # Tests du cross-LLM review
│   │   └── mocks/
│   │       ├── shopify_responses.py
│   │       ├── stripe_disputes.py
│   │       ├── meta_insights.py
│   │       ├── claude_responses.py
│   │       └── gpt_gemini_responses.py
│   │
│   ├── pyproject.toml
│   ├── Dockerfile                      # API service
│   ├── Dockerfile.worker               # Celery worker service
│   └── requirements.txt
│
├── frontend/
│   ├── src/
│   │   ├── app/                        # Next.js App Router
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx                # Landing page (SSR, SEO)
│   │   │   ├── dashboard/
│   │   │   │   ├── layout.tsx          # Dashboard layout, navigation onglets
│   │   │   │   ├── page.tsx            # Redirect vers /dashboard/profit
│   │   │   │   ├── profit/page.tsx     # Onglet Profit (P&L, LTV, margins, FX)
│   │   │   │   ├── fraud/page.tsx      # Onglet Anti-Fraude (scores, holds, card testing, FP tracker)
│   │   │   │   ├── chargebacks/page.tsx # Onglet Chargebacks (evidence approval UI)
│   │   │   │   ├── ads/page.tsx        # Onglet Ads (spend 100%, integration health)
│   │   │   │   ├── tariffs/page.tsx    # Onglet Landed Cost (Pro/Enterprise)
│   │   │   │   ├── reports/page.tsx    # Weekly reports + tax export
│   │   │   │   └── settings/page.tsx   # Settings (alerts, integrations, plan)
│   │   │   ├── admin/
│   │   │   │   └── agents/page.tsx     # Supervisor dashboard (internal only)
│   │   │   ├── onboarding/page.tsx     # Shopify install + ads connect + first P&L
│   │   │   └── pricing/page.tsx        # Plans + upgrade
│   │   │
│   │   ├── components/
│   │   │   ├── ui/                     # shadcn/ui components
│   │   │   ├── profit/                 # PnLChart, LTVCurve, MarginCard, FXLossCard
│   │   │   ├── fraud/                  # RiskScore, HoldQueue, CardTestingAlert, FalsePositiveCard
│   │   │   ├── chargebacks/            # EvidenceViewer, ApprovalFlow, DisputeList
│   │   │   ├── ads/                    # AdSpendChart, IntegrationBadge
│   │   │   ├── admin/                  # AgentStatusPanel, BudgetChart, TaskTimeline
│   │   │   └── shared/                 # AlertBanner, LoadingState, ErrorState
│   │   │
│   │   ├── lib/
│   │   │   ├── api.ts                  # API client typed (fetch wrapper)
│   │   │   ├── supabase.ts             # Supabase browser client
│   │   │   └── utils.ts
│   │   │
│   │   ├── hooks/
│   │   │   ├── use-profit.ts
│   │   │   ├── use-fraud-score.ts
│   │   │   ├── use-chargebacks.ts
│   │   │   ├── use-ad-spend.ts
│   │   │   ├── use-agents.ts           # Supervisor dashboard data
│   │   │   └── use-subscription.ts
│   │   │
│   │   └── types/
│   │       ├── order.ts
│   │       ├── profit.ts
│   │       ├── fraud.ts
│   │       ├── chargeback.ts
│   │       ├── agent.ts                # Agent run types pour admin dashboard
│   │       └── api.ts
│   │
│   ├── public/
│   ├── next.config.js
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   └── package.json
│
├── database/
│   ├── schema.sql                      # Schéma complet (source de vérité = docs/DATABASE.md)
│   └── migrations/
│       └── 001_initial.sql
│
├── docs/                               # 26 fichiers — documentation technique détaillée
└── .claude/                            # Skills, commands, agents Claude Code
```

---

## PATTERNS OBLIGATOIRES

### Python — Backend

```python
# 1. TOUJOURS Pydantic v2 pour la validation
from pydantic import BaseModel, Field

class OrderProfitRequest(BaseModel):
    order_id: str = Field(..., min_length=1)
    include_landed_cost: bool = Field(default=True)

# 2. TOUJOURS l'injection de dépendances FastAPI
from app.dependencies import get_supabase, get_shopify_client, get_stripe_disputes

@router.post("/profit/order/{order_id}")
async def get_order_profit(
    order_id: str,
    supabase: SupabaseClient = Depends(get_supabase),
    shopify: ShopifyClient = Depends(get_shopify_client),
):
    ...

# 3. TOUJOURS AppError, JAMAIS HTTPException directement
from app.core.exceptions import AppError, ErrorCode

raise AppError(
    code=ErrorCode.PROFIT_CALCULATION_FAILED,
    message="Cannot calculate profit: missing COGS for SKU-1234",
    status_code=422,
    context={"order_id": order_id, "missing_sku": "SKU-1234"},
)

# 4. TOUJOURS structlog, JAMAIS print() ou logging.info()
import structlog
logger = structlog.get_logger()
logger.info("profit_calculated", order_id=order_id, net_profit_cents=net_cents)

# 5. TOUJOURS typer les retours
async def calculate_true_profit(order_id: str) -> OrderProfit | None:
    ...

# 6. TOUJOURS async pour les I/O
async def fetch_ad_spend(integration_id: str, date: date) -> Decimal:
    ...

# 7. TOUJOURS httpx (async), JAMAIS requests (sync)
async with httpx.AsyncClient() as client:
    response = await client.get(url, headers=headers)

# 8. TOUJOURS Decimal pour l'argent, JAMAIS float
from decimal import Decimal
revenue: Decimal = Decimal("99.99")  # correct
revenue_wrong: float = 99.99          # INTERDIT — erreurs d'arrondi

# 9. TOUJOURS stocker l'argent en cents (INTEGER) en DB
#    Conversion Decimal → cents : int(amount * 100)
#    Conversion cents → Decimal : Decimal(cents) / Decimal(100)

# 10. TOUJOURS charger la Constitution de l'agent en system prompt
from app.agent.constitutions_loader import load_constitution

system_prompt = load_constitution("fraud_investigator")  # lit fraud_investigator.md
messages = [{"role": "system", "content": system_prompt}, ...]

# 11. Décision high-stakes → TOUJOURS cross-LLM review
from app.agent.cross_llm_review import review_decision

primary_decision = await claude_generate(...)
review = await review_decision(
    decision=primary_decision,
    stakes="high",  # evidence >500$, hold >1000$, margin alert critique
    reviewers=["gpt-5", "gemini-2.5"],
)
if review.consensus is False:
    # Escalade humaine obligatoire
    await escalate_to_human(decision=primary_decision, review=review)
```

### TypeScript — Frontend

```typescript
// 1. JAMAIS de `any`. Utiliser `unknown` ou type explicite.
// ❌ const data: any = await response.json()
// ✅ const data = await response.json() as OrderProfit

// 2. TOUJOURS typer les props
interface RiskScoreProps {
  score: number;        // 0-100
  level: "low" | "medium" | "high";
  reasons: string[];
}
export function RiskScore({ score, level, reasons }: RiskScoreProps) { ... }

// 3. TOUJOURS le API client centralisé
import { api } from "@/lib/api";
const profit = await api.profit.getOrder(orderId);

// 4. TOUJOURS gérer loading/error
const { data, isLoading, error } = useProfit(orderId);
if (isLoading) return <Skeleton />;
if (error) return <ErrorState message={error.message} />;

// 5. Affichage monétaire : TOUJOURS formater via Intl.NumberFormat
const formatted = new Intl.NumberFormat(locale, {
  style: "currency",
  currency: merchantCurrency,
}).format(cents / 100);
```

### Supabase

```sql
-- TOUJOURS RLS sur chaque table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY "merchants_own_orders" ON orders
    FOR ALL USING (merchant_id = auth.uid());

-- TOUJOURS NOTIFY après changement de schéma
NOTIFY pgrst, 'reload schema';

-- TOUJOURS indexer les colonnes de filtrage fréquent
CREATE INDEX idx_orders_merchant_id ON orders(merchant_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_fraud_scores_risk_level ON fraud_scores(risk_level) WHERE risk_level IN ('high', 'critical');
```

---

## INTERDICTIONS

### Python
- ❌ `HTTPException` directement → ✅ `AppError` avec `ErrorCode`
- ❌ `print()` → ✅ `structlog.get_logger()`
- ❌ `requests` (sync) → ✅ `httpx` (async)
- ❌ `float` pour l'argent → ✅ `Decimal` en code, `INTEGER cents` en DB
- ❌ `datetime.now()` → ✅ `datetime.now(UTC)` (toujours UTC)
- ❌ `os.getenv()` inline → ✅ `config.py` via Pydantic `BaseSettings`
- ❌ `try: ... except Exception:` → ✅ Catch exceptions spécifiques
- ❌ SQL brut sans paramètres → ✅ Parameterized queries
- ❌ Secrets en dur dans le code → ✅ Env vars via `config.py`
- ❌ `from typing import Optional` → ✅ `str | None` (Python 3.12+)
- ❌ Token Shopify/Stripe/Meta en clair en DB → ✅ Fernet encryption
- ❌ **Soumettre evidence chargeback sans approbation humaine** → ✅ Workflow approval obligatoire
- ❌ **Invoquer un agent spécialiste sans charger sa Constitution** → ✅ `constitutions_loader.load_constitution(agent_id)`
- ❌ **Décision high-stakes sans cross-LLM review** → ✅ `cross_llm_review.review_decision()`
- ❌ **Code agent qui se modifie lui-même** (self-modification patterns) → ✅ Le code des agents est immuable en runtime. Retraining ML et Mem0 updates oui, self-code-modification non.

### TypeScript
- ❌ `any` → ✅ Types explicites ou `unknown`
- ❌ `console.log` en production → ✅ Logger structuré
- ❌ `fetch()` inline partout → ✅ `api` client dans `lib/api.ts`
- ❌ CSS inline ou modules CSS → ✅ Tailwind uniquement
- ❌ `var` → ✅ `const` / `let`
- ❌ `enum` → ✅ `as const` objects ou union types
- ❌ `Number(amount)` / math flottante pour l'argent → ✅ cents INTEGER, format à l'affichage

### Architecture
- ❌ Logique métier dans les routes API → ✅ Services layer
- ❌ Appel Shopify/Stripe/Meta depuis le frontend → ✅ Toujours via le backend
- ❌ Agent spécialiste qui appelle directement un autre agent → ✅ Toujours passer par le Supervisor
- ❌ Modifier la DB sans migration numérotée → ✅ `database/migrations/`
- ❌ Commit sur `main` directement → ✅ Branch + PR
- ❌ Deploy sans tests qui passent → ✅ CI vérifie avant merge
- ❌ Stocker CVV / PAN complets → ✅ PCI DSS — on ne touche JAMAIS à ces données, Stripe les gère

---

## ERROR HANDLING

Hiérarchie centralisée. Catalogue complet dans `docs/ERRORS.md`.

```python
# app/core/exceptions.py

class AppError(Exception):
    """Base. TOUTES les erreurs héritent de celle-ci."""
    def __init__(self, code: ErrorCode, message: str, status_code: int = 500,
                 context: dict | None = None):
        self.code = code
        self.message = message
        self.status_code = status_code
        self.context = context or {}

class ShopifyError(AppError): ...      # API Shopify (rate limit, token expiré)
class StripeError(AppError): ...       # Stripe billing + disputes
class AdsAPIError(AppError): ...       # Meta / Google / TikTok
class ProfitError(AppError): ...       # Erreurs calcul profit
class FraudError(AppError): ...        # Erreurs scoring fraude
class ChargebackError(AppError): ...   # Erreurs workflow dispute
class IntegrationError(AppError): ...  # OAuth, disconnection, token refresh
class BillingError(AppError): ...      # Stripe subscription
class AuthError(AppError): ...         # Auth/permissions
class AgentError(AppError): ...        # Claude API, Mem0, LangGraph, Supervisor routing
class ConstitutionError(AppError): ... # Constitution file missing ou violation détectée
class ReviewError(AppError): ...       # Cross-LLM review échec (timeout, désaccord)
class IntegrityError(AppError): ...    # Data reconciliation mismatch
class FXError(AppError): ...           # FX rate fetch / conversion failure

# Handler global — main.py
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    logger.error("app_error", code=exc.code, message=exc.message, **exc.context)
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": exc.message}},
    )
```

---

## VARIABLES D'ENVIRONNEMENT

```bash
# === Supabase ===
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...     # Backend only, JAMAIS exposé au frontend

# === Shopify ===
SHOPIFY_API_KEY=xxx
SHOPIFY_API_SECRET=xxx
SHOPIFY_API_VERSION=2026-01
SHOPIFY_SCOPES=read_products,read_orders,read_customers,read_fulfillments,read_reports,read_inventory,read_locations

# === Stripe ===
STRIPE_SECRET_KEY=sk_...
STRIPE_PUBLISHABLE_KEY=pk_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_DISPUTES_WEBHOOK_SECRET=whsec_...

# === Meta Ads ===
META_APP_ID=xxx
META_APP_SECRET=xxx
META_API_VERSION=v21.0

# === Google Ads ===
GOOGLE_ADS_DEVELOPER_TOKEN=xxx
GOOGLE_ADS_CLIENT_ID=xxx
GOOGLE_ADS_CLIENT_SECRET=xxx

# === TikTok Ads ===
TIKTOK_APP_ID=xxx
TIKTOK_APP_SECRET=xxx

# === LLMs ===
ANTHROPIC_API_KEY=sk-ant-...         # Claude Opus (primary)
OPENAI_API_KEY=sk-...                # GPT-5 (cross-LLM review)
GOOGLE_AI_API_KEY=xxx                # Gemini 2.5 (cross-LLM review)

# === Cross-LLM Review config ===
CROSS_LLM_REVIEW_ENABLED=true
CROSS_LLM_STAKES_THRESHOLD_CENTS=50000   # $500 — seuil pour déclencher le review
CROSS_LLM_REVIEWERS=gpt-5,gemini-2.5     # Liste des reviewers
CROSS_LLM_TIMEOUT_SECONDS=30

# === Mem0 ===
MEM0_API_KEY=xxx                      # Si hosted. Sinon config pgvector dans Supabase.

# === Redis ===
REDIS_URL=redis://...                  # Railway managed Redis

# === Resend ===
RESEND_API_KEY=re_...

# === Sentry ===
SENTRY_DSN=https://...

# === LangSmith ===
LANGCHAIN_API_KEY=ls_...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=pilotprofit

# === FX rates (multi-currency) ===
FX_RATES_PROVIDER=openexchangerates    # ou fixer.io, exchangerate-api
FX_RATES_API_KEY=xxx

# === App ===
APP_ENV=development|staging|production
# Pre-launch URLs :
APP_URL=https://pilotprofit.vercel.app
BACKEND_URL=https://pilotprofit-api.up.railway.app
# Post-launch (après achat du domaine) :
# APP_URL=https://pilotprofit.app
# BACKEND_URL=https://api.pilotprofit.app

# === Encryption ===
FERNET_KEY=xxx                         # Chiffrement tokens OAuth

# === Blacklist (cross-merchant) ===
BLACKLIST_HASH_SALT=xxx                # Salt pour tokenization

# === Admin dashboard ===
ADMIN_DASHBOARD_ALLOWED_EMAILS=founder@pilotprofit.app,ops@pilotprofit.app

# === Recovery partners (optional) ===
TSI_API_KEY=xxx
ROCKET_API_KEY=xxx
```

Chargées via Pydantic `BaseSettings` dans `backend/app/config.py`.
Frontend : uniquement les vars préfixées `NEXT_PUBLIC_`.

---

## SHOPIFY SCOPES

```
read_products       → Product analytics, COGS, margin, SKU-level profit
read_orders         → Profit calc, fraud scoring, chargeback context
read_customers      → LTV, return abuse detection, customer cohort analysis
read_fulfillments   → Multi-fulfillment sync (ShipStation, ShipBob, MCF)
read_reports        → Shopify Reports API for data integrity reconciliation
read_inventory      → Inventory cost, dead stock tracking
read_locations      → Multi-warehouse COGS allocation
```

**Important :** PilotProfit est en READ-ONLY sur Shopify. Aucun write_*.
Raison : on ne modifie JAMAIS le store du merchant. On lit, on analyse, on alerte.

---

## API REST — ROUTES PRINCIPALES

```
# Auth
GET    /api/v1/auth/install              # Shopify OAuth redirect
GET    /api/v1/auth/callback             # Shopify OAuth callback
POST   /api/v1/auth/logout               # Logout

# Profit
GET    /api/v1/profit/pnl                # Daily P&L (paginated by date range)
GET    /api/v1/profit/orders/{order_id}  # True profit per order
GET    /api/v1/profit/products           # Profit per product (sortable)
GET    /api/v1/profit/ltv                # Clean LTV (excl. bots, non-buyers)
GET    /api/v1/profit/cashflow-forecast  # 30/60/90d forecast
GET    /api/v1/profit/margins/alerts     # Products under threshold
GET    /api/v1/profit/fx-loss            # Currency FX loss per period / per currency

# Fraud
GET    /api/v1/fraud/score/{order_id}    # Pre-Ship Score + reasons
GET    /api/v1/fraud/holds               # Orders on hold (pending review)
POST   /api/v1/fraud/holds/{id}/release  # Release hold (approve order)
POST   /api/v1/fraud/holds/{id}/reject   # Cancel order as fraud
GET    /api/v1/fraud/false-positive-cost # Net: savings vs lost revenue
GET    /api/v1/fraud/ratio               # Chargeback ratio vs Visa/MC thresholds
GET    /api/v1/fraud/card-testing/alerts # Active card testing attacks detected

# Chargebacks
GET    /api/v1/chargebacks                         # List disputes (paginated)
GET    /api/v1/chargebacks/{id}                    # Full dispute detail + draft evidence
POST   /api/v1/chargebacks/{id}/evidence/generate  # Build evidence PDF draft
POST   /api/v1/chargebacks/{id}/evidence/review    # Trigger cross-LLM review sur l'evidence
POST   /api/v1/chargebacks/{id}/evidence/approve   # Human approves — then submit to Stripe
POST   /api/v1/chargebacks/{id}/evidence/reject    # Human rejects — no submission
POST   /api/v1/chargebacks/{id}/recovery/send      # Send to TSI/Rocket

# Ads
GET    /api/v1/ads/spend                 # Total spend (all channels)
GET    /api/v1/ads/spend/by-channel      # Breakdown Meta/Google/TikTok
GET    /api/v1/ads/attribution           # Cross-platform attribution
GET    /api/v1/ads/integrations/health   # API connection status per channel

# Tariffs & Landed Cost
GET    /api/v1/tariffs/landed-cost/{sku} # Landed cost per SKU
POST   /api/v1/tariffs/simulate          # Simulate tariff change impact
GET    /api/v1/tariffs/duties-lost       # Non-refundable duties on returns

# Tax
GET    /api/v1/tax/export/quickbooks     # QuickBooks format
GET    /api/v1/tax/export/xero           # Xero format
GET    /api/v1/tax/export/csv            # Generic CSV

# Integrity
GET    /api/v1/integrity/checks          # Latest reconciliation reports
GET    /api/v1/integrity/alerts          # Data mismatch alerts (>2% variance)

# Integrations
POST   /api/v1/integrations/meta/connect       # OAuth initiation
POST   /api/v1/integrations/google/connect
POST   /api/v1/integrations/tiktok/connect
POST   /api/v1/integrations/quickbooks/connect
POST   /api/v1/integrations/xero/connect
DELETE /api/v1/integrations/{id}               # Disconnect

# Feedback (self-improving loop)
POST   /api/v1/feedback                        # Accept/reject recommendation

# Admin (Supervisor dashboard — internal only, IP-whitelist + email-allowlist)
GET    /api/v1/admin/agents                    # Agents status (all 5 specialists + supervisor)
GET    /api/v1/admin/agents/{agent_id}/runs    # Recent runs (paginated)
GET    /api/v1/admin/agents/{agent_id}/budget  # LLM budget per agent (from LangSmith)
GET    /api/v1/admin/cross-llm-reviews         # Recent cross-LLM reviews (consensus/disagreements)
GET    /api/v1/admin/escalations               # Decisions escalated to humans

# Billing
POST   /api/v1/billing/checkout                # Create Stripe checkout
GET    /api/v1/billing/portal                  # Stripe customer portal URL

# Notifications
GET    /api/v1/notifications                   # List (paginated)
PATCH  /api/v1/notifications/{id}/read         # Mark read

# Reports
GET    /api/v1/reports/weekly                  # Latest weekly report
GET    /api/v1/reports/monthly                 # Monthly summary

# Webhooks (pas de JWT — HMAC/signature validation)
POST   /api/v1/webhooks/shopify                # Shopify webhooks
POST   /api/v1/webhooks/stripe                 # Stripe billing webhooks
POST   /api/v1/webhooks/stripe-disputes        # Stripe disputes webhooks
POST   /api/v1/webhooks/meta                   # Meta Ads webhooks (spend updates)

# Healthcheck
GET    /api/v1/health                          # Status + DB + Redis + external APIs + LLMs
```

Versionné `/api/v1/`. JSON. Paginé cursor-based. Auth JWT sauf webhooks (HMAC) et admin (email-allowlist).

---

## WORKFLOW DE DÉVELOPPEMENT

### Ajouter une feature (end-to-end)

1. **Lire** `docs/FEATURES.md` — trouver la feature, son module, son plan requis, son agent responsable
2. **DB** — migration dans `database/migrations/`, RLS policy, `NOTIFY pgrst`
3. **Analyzer** — dans `backend/app/agent/analyzers/`, hérite de `BaseAnalyzer`
4. **Specialist** — si nécessaire, mettre à jour la Constitution de l'agent concerné dans `backend/app/agent/constitutions/`
5. **Service** — logique métier dans `backend/app/services/`
6. **Endpoint** — route dans `backend/app/api/routes/`, Pydantic schemas
7. **Frontend** — composant + page si nécessaire
8. **Tests** — pytest backend + vitest frontend
9. **Checklist** — `.claude/skills/feature-impl/SKILL.md`

### Ajouter un analyzer

Command `/add-analyzer` ou skill `.claude/skills/analysis-pipeline/SKILL.md`.

### Modifier une Constitution d'agent

Les Constitutions sont des contrats comportementaux. Toute modification :
1. Doit être discutée en PR avec review humaine obligatoire
2. Doit passer les tests dans `tests/test_supervisor/test_constitutions.py`
3. Ne doit JAMAIS affaiblir les règles de sécurité (approbation humaine, data integrity, PCI DSS)

### Deploy

Command `/deploy` ou `docs/DEPLOY.md`.

---

## CONVENTIONS

### Nommage

| Élément | Convention | Exemple |
|---------|-----------|---------|
| Fichiers Python | snake_case | `fraud_scorer.py` |
| Classes Python | PascalCase | `FraudScorer` |
| Fonctions Python | snake_case | `calculate_true_profit()` |
| Fichiers TS/TSX | kebab-case | `risk-score.tsx` |
| Composants React | PascalCase | `RiskScore` |
| Tables DB | snake_case pluriel | `orders`, `fraud_scores`, `chargebacks`, `agent_runs` |
| Colonnes DB | snake_case | `order_id`, `created_at`, `net_profit_cents` |
| Env vars | SCREAMING_SNAKE | `SUPABASE_URL` |
| Branches git | type/description | `feat/fraud-scorer`, `fix/profit-multi-currency` |
| Constitutions | snake_case.md | `fraud_investigator.md` |

### Commits

```
feat(fraud): add Pre-Ship Score rules engine
fix(profit): handle refunds in True Profit calculation
refactor(supervisor): extract cross-LLM review to separate module
docs(constitution): update fraud_investigator rules
test(chargebacks): add Stripe dispute webhook mock
chore(deps): bump langgraph to 0.4.x
```

---

## RÉFÉRENCES

| Tu travailles sur... | Lis d'abord |
|---------------------|-------------|
| N'importe quoi | `context.md` |
| Une feature | `docs/FEATURES.md` |
| Le Supervisor et le routing | `docs/SUPERVISOR_AGENT.md` |
| Une Constitution d'agent | `backend/app/agent/constitutions/{agent}.md` |
| Un calcul de profit | `docs/PROFIT_FORMULAS.md` + `.claude/skills/profit-calculation/SKILL.md` |
| Le fraud scoring | `docs/FRAUD_RULES.md` + `docs/ML_MODELS.md` + `.claude/skills/fraud-scoring/SKILL.md` |
| Un chargeback | `docs/CHARGEBACK_WORKFLOW.md` + `.claude/skills/chargeback-evidence/SKILL.md` |
| Le cross-LLM review | `docs/SUPERVISOR_AGENT.md` (section Cross-LLM Review) |
| Le schéma DB | `docs/DATABASE.md` + `.claude/skills/supabase-patterns/SKILL.md` |
| L'API Shopify | `docs/SHOPIFY.md` + `.claude/skills/shopify-api/SKILL.md` |
| Les APIs ads | `docs/INTEGRATIONS.md` + `.claude/skills/ads-apis-integration/SKILL.md` |
| Les webhooks | `docs/WEBHOOKS.md` |
| L'agent LangGraph | `docs/AGENT.md` + `.claude/skills/agent-loop/SKILL.md` |
| Un analyzer | `.claude/skills/analysis-pipeline/SKILL.md` |
| Le billing Stripe | `.claude/skills/stripe-billing/SKILL.md` |
| La sécurité | `docs/SECURITY.md` + `.claude/skills/owasp-security/SKILL.md` |
| Supabase / DB | `.claude/skills/supabase-patterns/SKILL.md` |
| Le frontend | `docs/UI.md` |
| Le deploy | `docs/DEPLOY.md` |
| Les tests | `docs/TESTS.md` |
| Un bug | `.claude/skills/systematic-debugging/SKILL.md` |
| Mem0 / mémoire | `.claude/skills/mem0-integration/SKILL.md` |
| Landed cost / tarifs | `.claude/skills/landed-cost-duties/SKILL.md` |
| Tax export | `.claude/skills/tax-export/SKILL.md` |
| Data integrity | `docs/DATA_INTEGRITY.md` + `.claude/skills/data-integrity-reconciliation/SKILL.md` |
| Le monitoring / alerting | `docs/MONITORING.md` |
| Les textes / copy | `docs/COPY.md` |
| Un bug en prod/staging | `.claude/skills/systematic-debugging/SKILL.md` + `.claude/skills/saas-debug-pipeline/SKILL.md` |

---

## RÈGLES ABSOLUES

1. **JAMAIS de code sans tests.** Chaque endpoint, analyzer, service : minimum 1 test happy path + 1 test error + 1 test edge case.
2. **JAMAIS de migration DB sans RLS.** Table créée = RLS policy dans le même fichier.
3. **JAMAIS d'appel Shopify/Stripe/Meta/Google API sans rate limit handling.** 429 → exponential backoff retry.
4. **JAMAIS de token OAuth en clair en DB.** Fernet encryption pour tous les tokens (Shopify, Meta, Google, TikTok, QuickBooks, Xero).
5. **JAMAIS de `any` en TypeScript.** Zéro tolérance.
6. **JAMAIS de logique métier dans les routes.** Routes → Services → DB/API.
7. **JAMAIS de commit sur `main`.** Feature branch → PR → merge.
8. **JAMAIS de float pour l'argent.** Decimal en code, cents INTEGER en DB.
9. **JAMAIS de submit d'evidence chargeback sans approbation humaine.** C'est notre différenciation #1 vs Chargeflow. Workflow : generate → cross-LLM review → human approve → submit.
10. **JAMAIS de stockage de PAN, CVV ou données carte.** PCI DSS. Stripe gère, on n'y touche pas.
11. **JAMAIS d'invocation d'un agent spécialiste sans charger sa Constitution.** `constitutions_loader.load_constitution(agent_id)` obligatoire.
12. **JAMAIS de décision high-stakes sans cross-LLM review.** Seuil configurable via `CROSS_LLM_STAKES_THRESHOLD_CENTS` (défaut 500$).
13. **JAMAIS d'agent qui modifie son propre code en runtime.** Pattern self-modifying incompatible SaaS financier. Le code est immuable en prod, seuls Mem0 et les modèles ML sont updated via feedback.
14. **JAMAIS d'appel direct entre agents spécialistes.** Tout routing passe par le Supervisor.
15. **TOUJOURS vérifier le plan merchant** avant feature payante. Mapping dans `docs/FEATURES.md` et `docs/PRICING_PLANS.md`.
16. **TOUJOURS structlog** avec context (merchant_id, order_id, agent_id, run_id, etc.).
17. **TOUJOURS UTC** pour les dates. Le frontend convertit en timezone locale.
18. **TOUJOURS Data Integrity Check** avant d'afficher des chiffres à l'utilisateur. Si écart > 2% avec la source, alerte au lieu d'afficher.
19. **TOUJOURS calibrer par merchant** : les seuils de fraude, les margin alerts, les patterns — pas de one-size-fits-all.
20. **TOUJOURS logger les décisions du Supervisor** dans `agent_runs` table (routing choices, specialist invoked, outcome, cross-LLM review result).
