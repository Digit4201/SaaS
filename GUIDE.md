# 🤖 Telegram AI Bot SaaS - Guide de Démarrage Complet

**Version:** 2.0 (LemonSqueezy Edition)  
**Date:** 2026-02-11  
**Auteur:** TechPartner Agent

---

## 📋 Table des Matières

1. [C'est quoi ce projet ?](#cest-quoi-ce-projet-)
2. [Prérequis](#prérequis)
3. [Installation rapide](#installation-rapide)
4. [Configuration](#configuration)
5. [Architecture](#architecture)
6. [Commandes du bot](#commandes-du-bot)
7. [Gestion des abonnements](#gestion-des-abonnements)
8. [Monitoring](#monitoring)
9. [Maintenance](#maintenance)
10. [FAQ](#faq)

---

## 🎯 C'est quoi ce projet ?

Un **bot Telegram IA** monétisé avec un système d'abonnement.

| Plan | Prix | Messages |
|------|------|----------|
| **Free** | 0€/mois | 50/mois |
| **Pro** | 9.99€/mois | 2,000/mois |
| **Business** | 29.99€/mois | Illimité |

### Caractéristiques

- 🤖 Bot Telegram alimenté par OpenAI (GPT-3.5/GPT-4)
- 💳 Paiements via **LemonSqueezy** (Stripe européen)
- 📊 Limites de messages par abonnement
- 🛡️ Sécurité renforcée (firewall, encryption, etc.)
- 📈 Monitoring avec Grafana/Prometheus

---

## 🧩 Prérequis

| Prérequis | Version minimum | Usage |
|-----------|-----------------|-------|
| **Docker** | 20.x | Conteneurisation |
| **Docker Compose** | 2.x | Orchestration |
| **Git** | 2.x | Versioning |

### Ce qu'il te faut aussi

- [x] Un serveur Linux (VPS Hetzner, OVH, etc.)
- [x] Un nom de domaine (optionnel mais recommandé)
- [x] Un bot Telegram (@BotFather)
- [x] Un compte LemonSqueezy
- [x] Une clé API OpenAI

---

## 🚀 Installation rapide

### Option 1: Installation automatique (recommandée)

```bash
# Se connecter au serveur
ssh user@ton-serveur

# Cloner le projet (si pas déjà fait)
cd /root/.openclaw/workspace/agents/techpartner

# Lancer l'installation
bash install.sh
```

L'installateur te demandera:
- Ton nom de domaine
- Le token Telegram
- Les clés API (LemonSqueezy, OpenAI)

### Option 2: Installation manuelle

```bash
# 1. Cloner le projet
git clone https://github.com/ton-repo/telegram-ai-saas.git
cd telegram-ai-saas

# 2. Configurer les variables
cp .env.example .env
nano .env

# 3. Lancer l'installation
docker-compose up -d

# 4. Migrer la base de données
docker-compose run --rm api npx prisma migrate deploy

# 5. Créer les plans
docker-compose run --rm api npx prisma db seed

# 6. Configurer les webhooks
./scripts/configure-webhooks.sh
```

---

## ⚙️ Configuration

### Variables d'environnement

Crée un fichier `.env` à la racine:

```env
# ═══════════════════════════════════════════════════════════════════
# DOMAINE
# ═══════════════════════════════════════════════════════════════════
DOMAIN=bot.mondomaine.com

# ═══════════════════════════════════════════════════════════════════
# TELEGRAM
# ═══════════════════════════════════════════════════════════════════
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_WEBHOOK_SECRET=$(openssl rand -hex 32)

# ═══════════════════════════════════════════════════════════════════
# LEMONSQUEEZY
# ═══════════════════════════════════════════════════════════════════
LEMON_API_KEY=ls_xxxxxxxxxxxxxxxxxxxxxxxx
LEMON_WEBHOOK_SECRET=whsec_xxxxxxxx
LEMON_STORE_ID=store_xxxxxxxx

# ═══════════════════════════════════════════════════════════════════
# OPENAI
# ═══════════════════════════════════════════════════════════════════
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ═══════════════════════════════════════════════════════════════════
# BASE DE DONNÉES
# ═══════════════════════════════════════════════════════════════════
POSTGRES_USER=telegram_ai
POSTGRES_PASSWORD=$(openssl rand -hex 32)
POSTGRES_DB=telegram_ai
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}

# ═══════════════════════════════════════════════════════════════════
# REDIS
# ═══════════════════════════════════════════════════════════════════
REDIS_URL=redis://redis:6379

# ═══════════════════════════════════════════════════════════════════
# SECRETS
# ═══════════════════════════════════════════════════════════════════
ENCRYPTION_KEY=$(openssl rand -hex 32)
SESSION_SECRET=$(openssl rand -hex 64)
JWT_SECRET=$(openssl rand -hex 64)
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                    │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌─────────────────┐ ┌───────────┐ ┌─────────────────┐
    │   CADDY 2       │ │  CADDY 2  │ │   CADDY 2       │
    │   (Frontend)    │ │  (API)    │ │   (Monitoring)  │
    └────────┬────────┘ └─────┬─────┘ └────────┬────────┘
             │                │                │
             │    ┌──────────┴──────────┐     │
             │    │                     │     │
             │    ▼                     ▼     │
             │  ┌─────────────────────────────────────┐
             │  │           API (Fastify)              │
             │  │  ├── Telegram Webhook Handler        │
             │  │  ├── LemonSqueezy Webhook Handler    │
             │  │  ├── Rate Limiting Middleware        │
             │  │  ├── Authentication                  │
             │  │  └── REST API                        │
             │  └─────────────────────────────────────┘
             │                     │
             │         ┌───────────┴───────────┐
             │         ▼                       ▼
             │  ┌─────────────┐         ┌─────────────┐
             │  │   WORKER    │         │   WORKER    │
             │  │  (BullMQ)   │         │  (IA)       │
             │  └─────────────┘         └─────────────┘
             │         │                       │
             │         └───────────┬───────────┘
             │                     ▼
             │              ┌─────────────┐
             │              │   REDIS     │
             │              │  (Queue)    │
             │              └─────────────┘
             │                     │
             └─────────────────────┼─────────────────────┐
                                   │
                                   ▼
                          ┌─────────────────┐
                          │   POSTGRESQL    │
                          │  (Primary + HA) │
                          └─────────────────┘
```

### Services Docker

| Service | Image | Port | Rôle |
|---------|-------|------|------|
| **caddy** | caddy:2-alpine | 80, 443 | Reverse proxy + TLS |
| **api** | build:./api | 3000 | API REST + Webhooks |
| **worker** | build:./worker | 3001 | Traitement IA |
| **postgres** | postgres:15 | 5432 | Base de données |
| **redis** | redis:7-alpine | 6379 | Queue + Cache |

---

## 💬 Commandes du bot

Les utilisateurs peuvent taper ces commandes sur Telegram:

| Commande | Description |
|----------|-------------|
| `/start` | Créer un compte / voir les plans |
| `/usage` | Afficher le quota de messages restant |
| `/subscribe` | Voir les plans disponibles |
| `/pro` | Acheter le plan Pro (9.99€/mois) |
| `/business` | Acheter le plan Business (29.99€/mois) |
| `/cancel` | Annuler l'abonnement |
| `/help` | Afficher l'aide |

### Exemple de `/usage`

```
📊 Utilisation de Pierre

📦 Plan: Pro
📈 Statut: Actif

📝 Messages ce mois: 1,247/2,000
🟩🟩🟩🟩🟩⬜⬜⬜⬜⬜ 62%

🔄 Renouvellement: 15 mars 2026
```

---

## 💳 Gestion des abonnements

### Créer les plans sur LemonSqueezy

1. **Créer 3 variants dans LemonSqueezy:**

   | Nom | Prix | Fréquence |
   |-----|------|-----------|
   | Free | 0€ | Une fois |
   | Pro | 9.99€ | Mensuel |
   | Business | 29.99€ | Mensuel |

2. **Noter les Variant IDs** et les mettre à jour dans la base:
   ```bash
   docker-compose run --rm api npx prisma studio
   ```

3. **Configurer le webhook:**
   - URL: `https://ton-domaine/lemonsqueezy/webhook`
   - Secret: Copier `LEMON_WEBHOOK_SECRET` du .env

### Événements gérés

| Événement | Action |
|-----------|--------|
| `subscription_created` | Activer l'abonnement |
| `subscription_updated` | Mettre à jour le statut |
| `subscription_cancelled` | Annuler à la fin de la période |
| `subscription_payment_failed` | Notifier l'utilisateur |

---

## 📊 Monitoring

### Accéder à Grafana

```
URL: http://localhost:3000
Login: admin
Mot de passe: admin
```

### Dashboards disponibles

| Dashboard | Description |
|-----------|-------------|
| **Infrastructure** | CPU, RAM, Disque, Réseau |
| **Business Metrics** | MRR, Utilisateurs, Conversions |
| **API Performance** | Latence, Erreurs, Requêtes |

### Métriques clés

```promql
# Revenue Mensuel Récurrent
sum(lemon_payment_succeeded_amount)

# Utilisateurs actifs
count(distinct(user_id) by day)

# Taux d'erreur API
sum(rate(http_errors[5m])) / sum(rate(http_requests[5m]))
```

---

## 🔧 Maintenance

### Commandes usuelles

```bash
# Voir l'état des services
docker-compose ps

# Voir les logs
docker-compose logs -f
docker-compose logs -f api

# Redémarrer un service
docker-compose restart api

# Arrêter tout
docker-compose down

# Sauvegarder la base
./scripts/backup.sh

# Restaurer une sauvegarde
./scripts/restore.sh data/backups/postgres_20260211_120000.dump.gz
```

### Rotation des secrets

```bash
# Rotation tous les 90 jours
./scripts/rotate-secrets.sh --rotate
```

### Mise à jour

```bash
# Pull les dernières images
docker-compose pull

# Déploiement zero-downtime
./scripts/deploy.sh v1.2.0

# Rollback si problème
./scripts/deploy.sh --rollback
```

---

## ❓ FAQ

### Q: Combien ça coûte par mois ?

| Coût | Prix |
|------|------|
| Serveur (Hetzner CCX21) | ~5€/mois |
| Nom de domaine | ~10€/an |
| OpenAI (si usage intensif) | ~1-5€/mois |
| **Total** | **~10-15€/mois** |

### Q: Combien je gagne ?

| Scénario | Revenus/mois | Bénéfice/mois |
|----------|--------------|---------------|
| 100 users Pro | 999€ | ~985€ |
| 500 users Pro | 4,995€ | ~4,980€ |
| 1000 users Pro | 9,990€ | ~9,970€ |

### Q: Et si l'API OpenAI ne répond pas ?

Le système utilise un **fallback**:
1. Essaie GPT-4
2. Si échec → GPT-3.5-turbo
3. Si échec → Réponse d'erreur temporaire

### Q: Comment ajouter des admins ?

```typescript
// Dans la base de données
await prisma.user.update({
  where: { id: 'user-id' },
  data: { is_admin: true }
});
```

### Q: Le bot ne répond pas

```bash
# Vérifier les logs
docker-compose logs -f api | grep -i error

# Vérifier le webhook Telegram
curl https://api.telegram.org/botTOKEN/getWebhookInfo
```

---

## 📁 Structure du projet

```
telegram-ai-saas/
├── install.sh                    # Script d'installation automatique
├── docker-compose.yml            # Services Docker
├── .env                          # Variables d'environnement
├── Caddyfile                     # Configuration Caddy
│
├── api/                          # API principale
│   ├── Dockerfile
│   ├── prisma/
│   │   ├── schema.prisma         # Schéma base de données
│   │   └── seed.ts               # Données par défaut
│   └── src/
│       ├── index.ts              # Point d'entrée
│       ├── middleware/
│       │   └── rate-limit-user.ts # Vérification quotas
│       └── modules/
│           ├── telegram/
│           │   ├── commands.ts    # Commandes bot
│           │   └── webhook.handler.ts
│           └── billing/
│               ├── lemon-checkout.ts
│               └── webhook.handler.ts
│
├── worker/                       # Worker IA
│   ├── Dockerfile
│   └── src/
│       └── worker.ts             # Traitement messages
│
├── scripts/
│   ├── deploy.sh                 # Déploiement
│   ├── backup.sh                 # Sauvegarde
│   ├── restore.sh                # Restauration
│   └── rotate-secrets.sh         # Rotation secrets
│
├── monitoring/
│   ├── docker-compose.monitoring.yml
│   ├── prometheus.yml
│   └── grafana/
│       └── dashboards/
│           ├── infrastructure.json
│           └── business-metrics.json
│
└── projects/telegram-ai-saas/
    ├── ARCHITECTURE.md           # Doc technique
    ├── MODEL-ABONNEMENT.md       # Doc abonnement
    └── README.md                 # Ce fichier
```

---

## 🔗 Liens utiles

- [Telegram BotFather](https://t.me/BotFather)
- [LemonSqueezy Docs](https://docs.lemonsqueezy.com)
- [OpenAI API](https://platform.openai.com)
- [grammY Docs](https://grammy.dev)
- [Prisma Docs](https://prisma.io/docs)

---

## 📝 Licence

MIT License - Utilisation libre pour tes projets.

---

*Document généré par TechPartner Agent*
*11 février 2026*
