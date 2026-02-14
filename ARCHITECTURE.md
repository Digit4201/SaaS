# 🏗️ Architecture Telegram AI Bot SaaS - LemonSqueezy

**Version:** 2.0 (LemonSqueezy)  
**Date:** 2026-02-11  
**Status:** Production-Ready

---

## 📋 Table des Matières

1. [Vue d'Ensemble](#1-vue-densemble)
2. [Architecture Système](#2-architecture-système)
3. [Stack Technique](#3-stack-technique)
4. [Base de Données](#4-base-de-données)
5. [Sécurité](#5-sécurité)
6. [Déploiement](#6-déploiement)
7. [Monitoring](#7-monitoring)
8. [Disaster Recovery](#8-disaster-recovery)

---

## 1. Vue d'Ensemble

### 🎯 Objectif
Bot Telegram IA monétisé via abonnement (SaaS)

### 💰 Modèle Économique
| Plan | Prix | Quotas |
|------|------|--------|
| **Promo** | **4.99€/mois** | Illimité |
| Standard | 9.99€/mois | Illimité |

*Promo à vie pour les premiers utilisateurs!*

### 🏢 Infrastructure
- **VPS:** Hetzner Cloud (CCX62: 32GB RAM, 8 vCPU)
- **DB:** PostgreSQL 15 + Redis 7
- **Proxy:** Caddy 2 (TLS automatique)
- **Payment:** LemonSqueezy ( substitut Stripe )

---

## 2. Architecture Système

```
┌─────────────────────────────────────────────────────────────┐
│                      INTERNET                               │
└────────────────────────┬────────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │      CADDY 2        │  (TLS + Rate Limit L7)
              │   :80/:443          │
              └──────────┬──────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
┌────────▼────────┐ ┌────▼────┐ ┌───────▼───────┐
│   API v1.2      │ │ Worker  │ │   Worker      │
│   (Fastify)     │ │  (AI)   │ │   (AI)        │
│   Instance 1    │ │  x2     │ │   Instance 2  │
└────────┬────────┘ └────┬────┘ └───────┬───────┘
         │               │              │
         │    ┌──────────▼──────────┐   │
         │    │       REDIS         │   │
         │    │   (Queue + Cache)   │◄──┘
         │    └──────────┬──────────┘
         │               │
         └───────────────┼───────────────┘
                         │
              ┌──────────▼──────────┐
              │     POSTGRES        │
              │   (Primary + Hot    │
              │    Standby)         │
              └─────────────────────┘
                         │
              ┌──────────▼──────────┐
              │     S3/MINIO        │
              │  (Backups + Files)  │
              └─────────────────────┘
```

### Flux de données

```mermaid
graph TD
    A[Telegram User] --> B[Caddy Webhook]
    B --> C{API Instance}
    C --> D[Validate Signature]
    D --> E[Redis Rate Limit]
    E --> F[Postgres - Save Message]
    F --> G[Redis Queue]
    G --> H[Worker AI]
    H --> I[OpenAI API]
    I --> J[Update Message]
    J --> K[Telegram Response]
    
    L[LemonSqueezy Webhook] --> C
    M[Cron Job] --> N[Backup S3]
    N --> O[Verify Backup]
```

---

## 3. Stack Technique

| Composant | Technologie | Raison |
|-----------|-------------|--------|
| **API Server** | Fastify | Perfs 5x Express, async native |
| **Bot Telegram** | grammY | Meilleure DX que Telegraf |
| **ORM** | Prisma | Type-safe, migrations simples |
| **Queue** | BullMQ | Robust, retry built-in |
| **BDD** | PostgreSQL 15 | Relationnel, RLS natif |
| **Cache/Queue** | Redis 7 | Rapide, persistent AOF |
| **Reverse Proxy** | Caddy 2 | TLS auto, config simple |
| **Payment** | LemonSqueezy | Stripe-like, international |
| **IA** | OpenAI GPT-4/3.5 | Qualité/prix |
| **Monitoring** | Prometheus + Grafana | Complet, gratuit |
| **Logs** | Pino + Loki | Structuré |

### Schéma des URLs

```
https://api.tondomaine.com/
├── /telegram/webhook          # Webhook Telegram (POST)
├── /lemonsqueezy/webhook      # Webhook Paiements (POST)
├── /health                    # Health check (GET)
├── /ready                     # Readiness probe (GET)
├── /metrics                   # Prometheus (GET)
├── /export                    # RGPD export (GET, auth)
├── /delete                    # RGPD delete (DELETE, auth)
└── /admin/*                   # Dashboard admin (auth)
```

---

## 4. Base de Données

### Schéma Complet (15 tables)

```prisma
// Voir prisma/schema.prisma complet
```

### Relations

```
User (1) ───> (N) Subscription
User (1) ───> (N) Message
User (1) ───> (N) Session
User (1) ───> (N) AuditLog
User (1) ───> (N) SupportTicket
User (1) ───> (N) RateLimit
User (1) ───> (N) UserExperiment

Subscription (1) ───> (N) LemonSqueezyEvent
Plan (1) ───> (N) Subscription
Experiment (1) ───> (N) UserExperiment
FeatureFlag (1) ───> (N)
```

### Row-Level Security (RLS)

```sql
-- Politique RLS pour users
CREATE POLICY "Users can only see own data"
ON users FOR SELECT
USING (id = current_setting('app.current_user_id', true)::uuid);

-- Politique RLS pour messages
CREATE POLICY "Users can only see own messages"
ON messages FOR ALL
USING (user_id = current_setting('app.current_user_id', true)::uuid);
```

---

## 5. Sécurité

### 🔐 Authentification

```typescript
// JWT Token Structure
{
  "userId": "uuid",
  "sessionId": "uuid",
  "role": "user" | "admin",
  "exp": 1234567890,
  "iat": 1234567890
}
```

### 🛡️ Rate Limiting Granulaire

```typescript
// Rate limits par endpoint
const RATE_LIMITS = {
  '/telegram/webhook': { points: 100, duration: 60 },  // 100 req/min
  '/lemonsqueezy/webhook': { points: 10, duration: 60 }, // 10 req/min
  '/ai/generate': { points: 5, duration: 60 },           // 5 req/min
  '/export': { points: 3, duration: 3600 },              // 3 req/hour
};

// Par user (sliding window)
const USER_RATE_LIMITS = {
  free: { daily: 50, monthly: 0 },
  pro: { daily: 2000, monthly: 0 },
  business: { daily: 0, monthly: 0 }, // Illimité
};
```

### 🔑 Secrets Management

```bash
# Rotation automatique tous les 90 jours
# 1. Générer nouveau secret
NEW_SECRET=$(openssl rand -hex 32)

# 2. Double-run période (24h)
./scripts/rotate-secrets.sh --add $NEW_SECRET

# 3. Update webhooks LemonSqueezy
curl -X PUT "https://api.lemonsqueezy.com/v1/webhooks/$WEBHOOK_ID" \
  -H "Authorization: Bearer $LEMON_API_KEY" \
  -d '{"data": {"attributes": {"secret": "'$NEW_SECRET'"}}}'

# 4. Retirer ancien secret après 24h
./scripts/rotate-secrets.sh --remove $OLD_SECRET
```

### 🛡️ Content Security Policy

```typescript
// Fastify Helmet configuration
await fastify.register(fastifyHelmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.openai.com'],
    },
  },
});
```

---

## 6. Déploiement

### Zero-Downtime Deployment

```bash
# Déploiement Blue-Green
./scripts/deploy.sh v1.2.0

# Flow:
# 1. Build nouvelle image
# 2. Deploy instance 2 (v1.2)
# 3. Health check
# 4. Gradual traffic shift (10% → 50% → 100%)
# 5. Monitor error rate
# 6. Si > 0.5%, rollback automatique
```

### Variables d'Environnement

```env
# Requis
TELEGRAM_BOT_TOKEN=bot123:abc...
LEMON_API_KEY=ls_xxx
LEMON_WEBHOOK_SECRET=whsec_xxx
OPENAI_API_KEY=sk-xxx
DATABASE_URL=postgresql://user:pass@localhost:5432/db
REDIS_URL=redis://localhost:6379

# Secrets (via env files)
ENCRYPTION_KEY=openssl rand -hex 32
SESSION_SECRET=openssl rand -hex 64
JWT_SECRET=openssl rand -hex 64

# Optionnel
LOG_LEVEL=info
SENTRY_DSN=https://xxx@sentry.io/xxx
```

### Docker Services

| Service | Image | Resources | Restart |
|---------|-------|-----------|---------|
| caddy | caddy:2-alpine | 100MB | unless-stopped |
| api | build:./api | 1GB | unless-stopped |
| worker | build:./worker | 2GB | unless-stopped |
| redis | redis:7-alpine | 256MB | unless-stopped |
| postgres | postgres:15-alpine | 1GB | unless-stopped |

---

## 7. Monitoring

### 📊 Métriques Business (Grafana)

```promql
# MRR Mensuel
sum(lemon_payment_succeeded_amount)

# Churn Rate
sum(rate(subscriptions_canceled[7d])) / sum(rate(subscriptions_active[7d]))

# Utilisateurs Actifs Daily
count(distinct(user_id) by day)

# Messages/Jour
count(messages created_at > now() - 24h)

# Taux de conversion
sum(rate(checkout_completed[7d])) / sum(rate(checkout_started[7d]))
```

### 🚨 Alerts

| Alert | Condition | Sévérité |
|-------|-----------|----------|
| API Down | health check failed | Critical |
| High Error Rate | error_rate > 5% | Critical |
| DB Slow | query_time > 1s | Warning |
| Payment Failed | payment_failed > 10% | Warning |
| Churn Spike | churn > 20% | Warning |

---

## 8. Disaster Recovery

### 📋 RPO/RTO

| Scénario | RPO (Data Loss) | RTO (Recovery Time) |
|----------|-----------------|---------------------|
| DB Crash | < 1 minute | < 5 minutes |
| VPS Failover | < 5 minutes | < 10 minutes |
| Complete Disaster | < 1 hour | < 2 hours |

### 🔄 Procédure de Backup

```bash
# Backup PostgreSQL every hour
0 * * * * /app/scripts/backup.sh

# Backup Redis every 15 minutes
*/15 * * * * /app/scripts/backup-redis.sh

# Test restore monthly
0 3 1 * * /app/scripts/test-restore.sh
```

### 🆘 Procédure de Restauration

```bash
# 1. Arrêter les services
docker-compose stop api worker

# 2. Restaurer PostgreSQL
pg_restore -h postgres -U user -d db /backups/latest.dump

# 3. Restaurer Redis
redis-cli BGSAVE
redis-cli LASTSAVE

# 4. Redémarrer les services
docker-compose start

# 5. Vérifier
curl https://api.tondomaine.com/health
```

---

## 📁 Fichiers du Projet

```
projects/telegram-ai-saas/
├── ARCHITECTURE.md           # Ce document
├── docker-compose.yml        # Services production
├── prisma/
│   ├── schema.prisma         # DB schema complet
│   └── migrations/           # Migrations
├── scripts/
│   ├── deploy.sh             # Déploiement zero-downtime
│   ├── backup.sh             # Backup PostgreSQL
│   ├── restore.sh            # Restauration
│   ├── rotate-secrets.sh     # Rotation secrets
│   └── test-restore.sh       # Test restore
├── api/
│   ├── Dockerfile
│   ├── src/
│   │   ├── index.ts          # Fastify entry
│   │   ├── config/           # Env, DB, Redis
│   │   ├── modules/          # Telegram, AI, Billing
│   │   └── middleware/       # Auth, RLS, Errors
│   └── package.json
├── worker/
│   ├── Dockerfile
│   ├── src/
│   │   ├── queue.ts          # BullMQ
│   │   ├── ai.ts             # OpenAI processing
│   │   └── handlers/         # Job handlers
│   └── package.json
└── monitoring/
    ├── docker-compose.monitoring.yml
    ├── prometheus.yml
    └── grafana/
        ├── dashboards/
        └── provisioning/
```

---

*Document généré par TechPartner Agent*
*Telegram AI Bot SaaS v2.0 - LemonSqueezy*
*11 février 2026*
