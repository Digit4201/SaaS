# 🤖 Telegram AI Bot SaaS - Documentation Complète

**Version:** 2.0  
**Date:** 2026-02-12  
**Status:** Production-Ready

---

## 📋 Table des Matières

1. [Aperçu](#-aperçu)
2. [Fonctionnalités](#-fonctionnalités)
3. [Architecture](#-architecture)
4. [Installation Rapide](#-installation-rapide)
5. [Configuration](#-configuration)
6. [API Reference](#-api-reference)
7. [Monitoring](#-monitoring)
8. [Dépannage](#-dépannage)
9. [Sécurité](#-sécurité)

---

## 🎯 Aperçu

Bot Telegram IA monétisé via abonnements avec **LemonSqueezy**.

### Modèle Hybride BYOAI

| Service | Source |
|---------|--------|
| IA (Ollama local) | Admin (gratuit) |
| OpenAI | Client (BYOAI) |
| Claude Code | Client (BYOAI) |
| Brave Search | Client (BYOAI) |

---

## ✨ Fonctionnalités

### Core
- 🤖 Bot Telegram (grammY)
- 🧠 IA Multi-provider (Ollama, OpenAI, Claude)
- 💬 Conversations contextuelles

### Billing (LemonSqueezy)
- 💳 Abonnements mensuels/annuels
- 🔄 Webhooks idempotents

### SaaS
- 🎯 Rate limiting
- 🚦 Feature flags
- 🧪 A/B Testing

### RGPD
- 📤 Export données
- 🗑️ Suppression compte
- ⏰ Data retention

---

## 🏗️ Architecture

```
                    INTERNET
                        │
           ┌────────────▼────────────┐
           │       CADDY 2           │  (TLS + Rate Limit)
           │       :80/:443          │
           └────────────┬────────────┘
                        │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
┌───▼────┐     ┌──────▼──────┐   ┌─────▼─────┐
│  API   │     │   Worker    │   │  Portal   │
│ Fastify│     │   BullMQ    │   │  React    │
└───┬────┘     └──────┬──────┘   └─────┬─────┘
    │                  │                │
    └──────────────────┼────────────────┘
                       │
           ┌──────────▼──────────┐
           │       REDIS          │  (Queue + Cache)
           └──────────┬──────────┘
                      │
           ┌──────────▼──────────┐
           │      POSTGRES       │  (Primary)
           └─────────────────────┘
```

---

## 🚀 Installation Rapide

### 1. Prérequis

```bash
# Docker & Docker Compose
curl -fsSL https://get.docker.com | sh
docker compose version

# Git clone
git clone <repo-url>
cd telegram-ai-saas
```

### 2. Générer Secrets

```bash
# Générer tous les secrets automatiquement
./scripts/generate-secrets.sh

# Cela crée le fichier .env avec:
# - TELEGRAM_WEBHOOK_SECRET
# - LEMON_WEBHOOK_SECRET  
# - POSTGRES_PASSWORD
# - ENCRYPTION_KEY
# - SESSION_SECRET
# - JWT_SECRET
```

### 3. Configurer Variables

```bash
# Éditer .env avec vos clés
nano .env

# Required:
# - TELEGRAM_BOT_TOKEN
# - LEMON_API_KEY
# - LEMON_STORE_ID
```

### 4. Démarrer

```bash
# Démarrer PostgreSQL
docker-compose up -d postgres

# Attendre que Postgres soit prêt
sleep 5

# Appliquer migrations
docker-compose run --rm api npx prisma migrate deploy

# Démarrer tous les services
docker-compose up -d
```

### 5. Vérifier

```bash
# Health check
curl https://api.tondomaine.com/health

# Logs
docker-compose logs -f api
```

---

## ⚙️ Configuration

### Variables Obligatoires

```env
# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_WEBHOOK_SECRET=$(openssl rand -hex 32)

# LemonSqueezy
LEMON_API_KEY=ls_xxxxxxxxxxxxxxxxxxxxxxxx
LEMON_WEBHOOK_SECRET=$(openssl rand -hex 32)
LEMON_STORE_ID=store_xxxxxxxxxxxxxxxxxx

# Database
POSTGRES_USER=telegram_ai
POSTGRES_PASSWORD=xxx
POSTGRES_DB=telegram_ai
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}

# Redis
REDIS_URL=redis://redis:6379

# Secrets (64 chars min)
ENCRYPTION_KEY=xxx
SESSION_SECRET=xxx
JWT_SECRET=xxx
```

### Variables Optionnelles

```env
# Monitoring
SENTRY_DSN=https://xxx@sentry.io/xxx
LOG_LEVEL=info

# MinIO (backups)
S3_BUCKET=telegram-ai-backups
S3_ENDPOINT=https://minio.local:9000
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=xxx
```

---

## 📡 API Reference

### Health

```bash
GET /health
Response: { "status": "ok", "timestamp": "..." }
```

### Users

```bash
# Créer/éditer utilisateur (Telegram webhook)
POST /telegram/webhook

# Profile utilisateur
GET /api/v1/users/me
PUT /api/v1/users/me
```

### Subscriptions

``` Plans disponibles
GET /api/v1/plans
Response:
{
  "data": [
    {
      "id": "uuid",
      "name": "free",
      "display_name": "Gratuit",
      "price_cents": 0,
      "monthly_limit": 0
    },
    {
      "id": "uuid", 
      "name": "pro",
      "display_name": "Pro",
      "price_cents": 499,
      "monthly_limit": 0
    }
  ]
}

# Current subscription
GET /api/v1/subscription

# Cancel subscription
DELETE /api/v1/subscription
```

### AI Usage

```bash
# Statistiques usage AI
GET /api/v1/ai/usage
Response:
{
  "data": {
    "total_tokens": 15000,
    "total_cost_cents": 0,
    "by_model": {
      "ollama": { "tokens": 10000, "cost": 0 },
      "openai": { "tokens": 5000, "cost": 0 }
    }
  }
}
```

### Billing (LemonSqueezy)

```bash
# Créer checkout session
POST /api/v1/billing/checkout
Body: { "plan_id": "uuid", "success_url": "..." }

# Portal session
POST /api/v1/billing/portal
Response: { "url": "https://portal..." }
```

### Webhooks

```bash
# LemonSqueezy webhook
POST /lemonsqueezy/webhook
Headers: X-Signature = sha256 HMAC

# Telegram webhook
POST /telegram/webhook
```

---

## 📊 Monitoring

### Dashboards

| Dashboard | Accès | Description |
|-----------|-------|-------------|
| Infrastructure | Grafana | CPU, RAM, DB, Redis |
| Business | Grafana | MRR, Churn, Users |

### Métriques Clés

```promql
# MRR mensuel
sum(lemon_payment_succeeded_amount)

# Utilisateurs actifs (MAU)
count(distinct(user_id))

# Taux d'erreur API
sum(rate(http_errors[5m])) / sum(rate(http_requests[5m]))

# Messages par jour
count(messages) by day
```

### Configuration Sentry

```env
SENTRY_DSN=https://xxx@sentry.io/xxx
```

### Alertes

| Condition | Action |
|-----------|--------|
| API Down | Slack + PagerDuty |
| Error rate > 5% | Slack |
| DB query > 1s | Slack |
| Payment failed > 10% | Email |

---

## 🛠️ Dépannage

### Problèmes Courants

#### 1. API ne démarre pas

```bash
# Vérifier logs
docker-compose logs api

# Erreur commune: Database not ready
# Solution: Attendre postgres
docker-compose ps
docker-compose restart api
```

#### 2. Webhook Telegram ne fonctionne pas

```bash
# Vérifier le token
curl -X POST "https://api.telegram.org/bot<TOKEN>/getMe"

# Vérifier webhook
curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo

# Erreur: SSL certificate
# Solution: Caddy génère automatiquement Let's Encrypt
```

#### 3. Paiements LemonSqueezy pas reçus

```bash
# Vérifier webhook endpoint
curl -I https://api.tondomaine.com/lemonsqueezy/webhook

# Vérifier signature
# Le webhook utilise X-Signature header

# Logs webhook
docker-compose logs api | grep lemon
```

#### 4. Pas de réponses IA

```bash
# Vérifier Ollama
curl http://ollama:11434/api/tags

# Si Ollama HS, vérifier le container
docker-compose logs ollama

# Utiliser clé OpenAI client (BYOAI)
# Pas de fallback admin
```

### Commandes Utiles

```bash
# Status services
docker-compose ps

# Logs en temps réel
docker-compose logs -f

# Redémarrer un service
docker-compose restart api

# Vérifier base de données
docker-compose exec postgres psql -U telegram_ai -c "SELECT count(*) FROM \"User\";"

# Vider Redis cache
docker-compose exec redis redis-cli FLUSHALL

# Health check complet
curl -s http://localhost:3000/health | jq .
```

### Logs Structurés

```bash
# Logs JSON pour debugging
docker-compose exec api cat /app/logs/app.log

# Chercher erreurs
docker-compose logs api 2>&1 | grep -i error
```

---

## 🔐 Sécurité

### Checklist Production

- [x] TLS via Caddy (Let's Encrypt)
- [x] Rate limiting (IP + User)
- [x] JWT avec expiration
- [x] Secrets chiffrés
- [x] Webhooks signés
- [x] Audit logs

### Secrets Rotation

```bash
# Rotation tous les 90 jours
./scripts/rotate-secrets.sh --rotate

# Vérifier date rotation
grep ROTATION /etc/cron.daily/
```

### Variables Sensibles

```env
# Ne JAMAIS commiter
.env
.env.local

# Le .env.example contient les TEMPLATES ONLY
```

---

## 📁 Structure du Projet

```
telegram-ai-saas/
├── api/                    # API Fastify
│   ├── src/
│   │   ├── index.ts       # Entry point
│   │   ├── config/        # Env, DB, Redis
│   │   └── modules/       # telegram, billing, ai, etc.
│   └── prisma/schema.prisma
│
├── worker/                 # BullMQ workers
│   └── src/
│
├── scripts/
│   ├── generate-secrets.sh # Générateur secrets
│   ├── deploy.sh          # Déploiement
│   ├── backup.sh          # Backup DB + Redis
│   └── restore.sh          # Restauration
│
├── frontend/               # Portal React (optionnel)
│
├── tests/                  # Tests unitaires
│
├── Caddyfile              # Reverse proxy
├── docker-compose.yml
└── README.md
```

---

## 🔗 Liens Utiles

- [LemonSqueezy Docs](https://docs.lemonsqueezy.com)
- [grammY Docs](https://grammy.dev)
- [Prisma Docs](https://prisma.io/docs)
- [BullMQ Docs](https://docs.bullmq.io)

---

*Documentation générée par Agent-CTO*
*12 février 2026*
