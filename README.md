# PilotProfit

> **See your true profit. Stop fraud before it ships. In minutes, not months.**

Agent IA santé financière complète pour merchants Shopify. 41 features, 4 modules, système multi-agents orchestré.

---

## TL;DR

Les merchants Shopify perdent **30 000 $/an** à cause de 4 douleurs non corrélées :
1. **Profit fantôme** — ils pensent gagner 48%, ils gagnent 23% (ad spend sous-compté, LTV gonflée, upsells cassés, FX loss non tracké)
2. **Chargebacks** — 800 $/mois en moyenne drainés, 4h/jour à assembler des preuves manuellement
3. **Compta manuelle** — 20h/mois de DIY, 5K $/an de déductions manquées, urgences fiscales 3K $
4. **Landed cost ignoré** — tarifs, duties, retours imports non-refondables

Les outils actuels traitent ces 4 problèmes en silos. **PilotProfit les unifie** dans un agent qui détecte, analyse, agit et apprend — avec workflow humain obligatoire avant toute action à fort impact.

---

## Infrastructure

### Pre-launch (actuellement)
- **Frontend** : https://pilotprofit.vercel.app (Vercel)
- **Backend API** : https://pilotprofit-api.up.railway.app (Railway)

### Post-launch (après achat du domaine)
- **Production** : https://pilotprofit.app
- **API** : https://api.pilotprofit.app

---

## Stack

### Backend
- Python 3.12+ · FastAPI · Pydantic v2 · Celery · Redis
- LangGraph (Supervisor multi-agents) · Mem0 (mémoire persistante) · scikit-learn/LightGBM (ML fraud)
- Claude Opus (LLM primaire) · GPT-5 + Gemini 2.5 (cross-LLM review)
- Deploy : Railway

### Frontend
- Next.js 14 · TypeScript strict · Tailwind · shadcn/ui · Recharts
- Shopify App Bridge + Polaris (embedded admin)
- Deploy : Vercel

### Database
- Supabase PostgreSQL · Row Level Security · pgvector (Mem0)

### Services externes
- Shopify Admin API (GraphQL 2026-01)
- Stripe Billing + Stripe Disputes API
- Meta Marketing API · Google Ads API · TikTok Business API
- QuickBooks · Xero · TSI/Rocket (recovery)
- Resend (email) · Sentry · LangSmith

---

## Architecture multi-agents

Un **Supervisor Agent** (LangGraph) route les tâches vers **5 agents spécialistes**, chacun régi par sa propre **Constitution** (fichier markdown non-négociable chargé en system prompt) :

| Agent | Responsabilité |
|-------|----------------|
| **Profit Analyst** | True profit, LTV propre, landed cost, cashflow forecast, FX loss |
| **Fraud Investigator** | Pre-Ship Score, patterns, card testing, False Positive tracking |
| **Chargeback Specialist** | Evidence building, workflow human-in-loop, recovery |
| **Data Integrity** | Reconciliation Shopify ↔ Stripe ↔ Meta ↔ notre DB |
| **Ads Sync** | Meta/Google/TikTok spend 100%, attribution, health |

### 3 principes d'architecture

1. **Supervisor Pattern** — Un brain LangGraph orchestre les spécialistes. Observable via `/admin/agents`.
2. **Agent Constitutions** — Chaque agent a son fichier `.md` chargé en system prompt à chaque invocation. Garantit la cohérence comportementale dans le temps.
3. **Cross-LLM Review** — Décisions high-stakes (evidence > 500$, hold > 1000$, alerts critiques) : Claude génère → GPT-5/Gemini review → consensus ou escalade humaine.

---

## Les 4 modules (41 features)

- **Module 1 — Profit** (15 features) : True Profit, App Cost Allocator, Daily P&L, Margin Alerts, Ad Spend 100%, LTV propre, Multi-Fulfillment Sync, Tax Ready, Cashflow Forecast, Data Integrity Check, **Currency FX Loss Tracker**…
- **Module 2 — Anti-Fraude** (18 features) : Pre-Ship Score, Smart Hold, Auto-Evidence Builder (avec approbation humaine), Ratio Monitor, Blacklist cross-merchants tokenisée, Friendly Fraud Detector, Freight Forwarder Detection, 3DS Recommender, **Card Testing Detector**…
- **Module 3 — Intelligence** (5 features) : False Positive Cost Tracker (exclusivité), Promo Planner avec risque fraude, Return Abuse Detector…
- **Module 4 — Tarifs & Landed Cost** (3 features) : Landed Cost Calculator, Tariff Impact Simulator, Non-Refundable Duty Tracker…

Spec complète : [`docs/FEATURES.md`](docs/FEATURES.md)

---

## Pricing

| Plan | Prix | Orders/mois | Use case |
|------|------|-------------|----------|
| **Free** | 0 $ | 50 | Découverte, très petits stores |
| **Starter** | 49 $/mois | 500 | Solo merchant 100K-500K $/an |
| **Pro** | 129 $/mois | 2 000 | Multi-store 500K-5M $/an |
| **Enterprise** | 249 $/mois | Illimité | Agence DTC, 5-30 stores clients |

Détail et gating : [`docs/PRICING_PLANS.md`](docs/PRICING_PLANS.md)

---

## Getting started (dev)

### Prérequis

- Python 3.12+
- Node.js 20+
- Docker (Redis local) ou Redis cloud
- Compte Supabase (gratuit suffit pour dev)
- Compte Shopify Partner (pour créer une app de dev)
- API keys : Anthropic, OpenAI, Google AI, Stripe (test mode), Meta, Google Ads, TikTok

### Installation

```bash
# Clone
git clone https://github.com/YOUR_ORG/pilotprofit.git
cd pilotprofit

# Backend
cd backend
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env       # Remplir les clés
alembic upgrade head       # Migrations DB

# Frontend
cd ../frontend
npm install
cp .env.example .env.local # Remplir les clés publiques

# Redis local (Docker)
docker run -d -p 6379:6379 --name pilotprofit-redis redis:7-alpine
```

### Lancer en dev

```bash
# Terminal 1 — API FastAPI
cd backend
uvicorn app.main:app --reload --port 8000

# Terminal 2 — Celery worker
cd backend
celery -A tasks.celery_app worker --loglevel=info

# Terminal 3 — Celery beat (crons)
cd backend
celery -A tasks.celery_app beat --loglevel=info

# Terminal 4 — Frontend Next.js
cd frontend
npm run dev
```

L'app tourne sur `http://localhost:3000` et l'API sur `http://localhost:8000`.

### Tests

```bash
# Backend (pytest)
cd backend
pytest                              # Tous les tests
pytest tests/test_profit/           # Un module
pytest -k "fraud_scorer"            # Pattern matching
pytest --cov=app --cov-report=html  # Coverage

# Frontend (vitest)
cd frontend
npm test
npm run test:e2e  # Playwright E2E
```

---

## Structure du repo

```
pilotprofit/
├── CLAUDE.md                # Contexte principal Claude Code (à lire en premier)
├── context.md               # Vision business complète
├── README.md                # Ce fichier
│
├── backend/                 # FastAPI + LangGraph + Celery
│   ├── app/
│   │   ├── agent/           # Supervisor + 5 specialists + Constitutions
│   │   ├── api/             # Routes REST + middleware + webhooks
│   │   ├── services/        # Shopify, Stripe, Meta, Google, TikTok clients
│   │   ├── models/          # Modèles Pydantic
│   │   └── core/            # Exceptions, security, logging
│   ├── tasks/               # Celery tasks
│   └── tests/               # pytest
│
├── frontend/                # Next.js 14 + TypeScript + Tailwind
│   ├── src/
│   │   ├── app/             # App Router (dashboard, admin, onboarding)
│   │   ├── components/      # UI par module
│   │   ├── hooks/           # React hooks
│   │   └── lib/             # API client, Supabase, utils
│   └── tests/               # vitest + Playwright
│
├── database/
│   ├── schema.sql           # Source of truth
│   └── migrations/          # Migrations numérotées
│
├── docs/                    # 26 fichiers — doc technique complète
│   ├── FEATURES.md          # Spec des 41 features
│   ├── PROFIT_FORMULAS.md   # Formules math exactes
│   ├── FRAUD_RULES.md       # Catalogue règles fraud scoring
│   ├── CHARGEBACK_WORKFLOW.md
│   ├── SUPERVISOR_AGENT.md  # Architecture multi-agents
│   ├── DATABASE.md
│   ├── API.md
│   ├── INTEGRATIONS.md
│   ├── DEPLOY.md
│   └── …
│
└── .claude/                 # Config Claude Code
    ├── skills/              # 18 skills (shopify-api, fraud-scoring, …)
    ├── agents/              # 2 sub-agents (profit-analyst, fraud-investigator)
    └── commands/            # 6 slash-commands (/add-analyzer, /deploy, …)
```

---

## Règles absolues (extraits critiques)

Les 20 règles complètes sont dans [`CLAUDE.md`](CLAUDE.md). Les 6 les plus importantes :

1. **Jamais de submit d'evidence chargeback sans approbation humaine.** Workflow : generate → cross-LLM review → human approve → submit.
2. **Jamais de float pour l'argent.** Decimal en code, INTEGER cents en DB.
3. **Jamais d'agent qui modifie son propre code en runtime.** Pattern self-modifying incompatible SaaS financier.
4. **Jamais d'invocation d'un agent spécialiste sans charger sa Constitution.**
5. **Jamais d'appel direct entre agents spécialistes.** Tout routing passe par le Supervisor.
6. **Toujours Data Integrity Check** avant d'afficher des chiffres à l'utilisateur. Écart > 2% → alerte, pas affichage.

---

## Concurrents et différenciation

| Concurrent | Gap exploité par PilotProfit |
|------------|------------------------------|
| **TrueProfit** | Données fausses documentées → notre Data Integrity Check quotidien + LTV sans bots |
| **BeProfit** | 15% du Google Ads spend seulement → notre Total Ad Spend 100% avec alerte écart > 10% |
| **Chargeflow** | Auto-submit sans approbation ("prompt ChatGPT") → notre workflow humain obligatoire + cross-LLM review |
| **NoFraud/Wyllo** | Trop sensible, bloque des orders légitimes → notre False Positive Cost Tracker (exclusivité) |
| **Lifetimely/AMP** | Données dégradées post-acquisition → tout refait correctement, multi-fulfillment natif |
| **GoProfit** | Smart Alerts statiques, pas de fraude intégrée → alerts dynamiques Profit + Fraude unifiées |
| **Aucun concurrent Shopify** | Currency FX Loss Tracker (exclusivité) + Card Testing Detector temps réel |

Analyse complète : [`docs/COMPETITIVE_POSITIONING.md`](docs/COMPETITIVE_POSITIONING.md)

---

## Validation terrain

- 55+ threads Reddit scrapés, 5000+ commentaires
- 660+ reviews concurrents analysées (Shopify App Store + G2)
- 22 concurrents directs et indirects étudiés
- Mastercard 2025 + Market Clarity data integrated

---

## Contributing

1. Fork + feature branch (`feat/xxx` ou `fix/xxx`)
2. Implement + tests (pytest backend + vitest frontend — **CI refuse le merge sans tests**)
3. Pour toute modification d'une Constitution d'agent : review humaine obligatoire en PR
4. Pour toute modification touchant le calcul de profit ou le fraud scoring : double review
5. Commits conventional (`feat(fraud):`, `fix(profit):`, `docs(constitution):`, …)
6. PR vers `main` — CI doit passer (tests + linting + type check)

Workflow détaillé : [`CLAUDE.md`](CLAUDE.md) section "Workflow de développement".

---

## Security

- Tous les tokens OAuth (Shopify, Meta, Google, TikTok, QuickBooks, Xero) sont chiffrés via Fernet avant stockage
- Aucune donnée carte ne transite par PilotProfit (PCI DSS compliance via Stripe)
- Row Level Security (RLS) sur toutes les tables Supabase
- HMAC validation sur tous les webhooks entrants
- Admin dashboard protégé par email-allowlist + IP-whitelist

Report de vulnérabilité : `security@pilotprofit.app` (actif après achat du domaine ; avant cela, via GitHub Security Advisory privé).

---

## License

Proprietary. All rights reserved.

---

## Liens

### Pre-launch
- **App (staging)** : https://pilotprofit.vercel.app
- **API** : https://pilotprofit-api.up.railway.app
- **API docs** : https://pilotprofit-api.up.railway.app/docs

### Post-launch (à venir)
- **Production** : https://pilotprofit.app
- **API prod** : https://api.pilotprofit.app
- **Shopify App Store** : https://apps.shopify.com/pilotprofit
- **Status page** : https://status.pilotprofit.app
- **Support** : `support@pilotprofit.app`

---

**Version** : 0.1.0 (pre-launch)
**Dernière mise à jour** : 2026-04-24
