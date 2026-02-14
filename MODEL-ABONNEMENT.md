# 📦 RÉSUMÉ DES FICHIERS CRÉÉS - MODÈLE ABONNEMENT

## 🎯 Objectif
Implémenter le système d'abonnement avec limites de messages (Free/Pro/Business) via LemonSqueezy.

---

## 📁 Fichiers Créés/Modifiés

### 1. `api/src/middleware/rate-limit-user.ts` (NOUVEAU)
**Rôle:** Vérifie si l'utilisateur peut envoyer un message selon son plan.

**Fonctions:**
- `checkUserLimit()` - Vérifie quota avant chaque message
- `incrementUsage()` - Incrémente le compteur après chaque message
- `getUserQuota()` - Récupère les infos de quota pour l'affichage

**Flow:**
```
Message utilisateur
       │
       ▼
checkUserLimit()
       │
   ┌───┴───┐
   │       │
Quota OK  Limite atteinte
   │       │
   ▼       ▼
IA        "Upgrade vers Pro"
```

---

### 2. `api/src/modules/telegram/commands.ts` (NOUVEAU)
**Rôle:** Gère les commandes Telegram (/usage, /subscribe, /pro, /business, etc.)

**Commandes:**
| Commande | Action |
|----------|--------|
| `/usage` | Affiche le quota restant |
| `/subscribe` | Affiche les plans disponibles |
| `/pro` | Lance checkout plan Pro |
| `/business` | Lance checkout plan Business |
| `/cancel` | Annule l'abonnement |
| `/help` | Affiche l'aide |

**Exemple de réponse `/usage`:**
```
📊 Utilisation de Pierre

📦 Plan: Pro
📈 Statut: Actif

📝 Messages ce mois: 1,247/2,000
🟩🟩🟩🟩🟩⬜⬜⬜⬜⬜ 62%

🔄 Renouvellement: 15 mars
```

---

### 3. `api/src/modules/telegram/webhook.handler.ts` (NOUVEAU)
**Rôle:** Point d'entrée pour tous les messages Telegram.

**Flow:**
```
Webhook Telegram
       │
       ▼
Vérifier si commande (/start, /help...)
       │
   ┌───┴───┐
   │       │
Commande  Message normal
   │       │
   ▼       ▼
Exécuter  checkUserLimit()
commande         │
           ┌─────┴─────┐
           │           │
      Limite OK   Limite atteinte
           │           │
           ▼           ▼
           IA      "Quota dépassé"
           │
           ▼
      incrementUsage()
           │
           ▼
      Réponse user
```

---

### 4. `api/src/modules/billing/lemon-checkout.ts` (NOUVEAU)
**Rôle:** Gère l'intégration LemonSqueezy (paiements).

**Fonctions:**
- `createLemonCheckout()` - Crée une page de paiement
- `handleLemonWebhook()` - Traite les événements de paiement
- `cancelSubscriptionAtPeriodEnd()` - Annule à la fin du mois

**Événements gérés:**
| Event | Action |
|-------|--------|
| `subscription_created` | Active l'abonnement |
| `subscription_updated` | Met à jour le statut |
| `subscription_cancelled` | Plan expirera à la fin |
| `subscription_payment_failed` | Informe l'utilisateur |

---

### 5. `api/src/modules/billing/webhook.handler.ts` (NOUVEAU)
**Rôle:** Reçoit et vérifie les webhooks LemonSqueezy.

**Sécurité:**
- Vérification de la signature HMAC-SHA256
- Idempotence (pas de doublons)
- Logging de tous les événements

---

### 6. `api/prisma/seed.ts` (NOUVEAU)
**Rôle:** Crée les données par défaut dans la base.

**Crée:**
| Plan | Prix | Messages |
|------|------|----------|
| Free | 0€ | 50/mois |
| Pro | 9.99€ | 2,000/mois |
| Business | 29.99€ | Illimité |

**Commande:**
```bash
npm run prisma:seed
```

---

### 7. `api/src/middleware/rate-limit-user.ts` (MIS À JOUR)
**Rôle:** Gestion complète des quotas.

**Nouvelles fonctionnalités:**
- Barre de progression visuelle
- Date de reset affichée
- Messages d'erreur personalisés
- Support illimité (Business)

---

## 🔄 Flux Complet d'un Message

```
1. User envoie: "Raconte-moi une blague"
                │
                ▼
2. Webhook reçoit le message
                │
                ▼
3. Créer/récupérer utilisateur
                │
                ▼
4. Vérifier si commande (/usage, /subscribe...)
                │
   ┌────────────┴────────────┐
   │                         │
   ▼                         ▼
Commande                  Message normal
   │                         │
   ▼                         ▼
Exécuter              Vérifier quota
                       (checkUserLimit)
                           │
              ┌────────────┼────────────┐
              │                         │
         Quota OK                   Quota dépassé
              │                         │
              ▼                         ▼
         Appeler IA                "Upgrade vers Pro"
         (OpenAI)                       │
              │                         │
              ▼                         ▼
         Sauvegarder               /subscribe
         réponse                   proposé
              │                         │
              ▼                         ▼
         Incrémenter               Fin
         compteur
              │
              ▼
         Réponse user
```

---

## 📊 Modèle de Données

```
┌─────────────────────────────────────────────────────────┐
│                      USER                               │
│  id, telegram_id, is_banned, created_at...             │
└────────────────────┬────────────────────────────────────┘
                     │ 1:1
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   SUBSCRIPTION                          │
│  id, plan_id, status (active/past_due/canceled),       │
│  current_period_end, cancel_at_period_end               │
└────────────────────┬────────────────────────────────────┘
                     │ 1:N
                     ▼
┌─────────────────────────────────────────────────────────┐
│                SUBSCRIPTION_USAGE                        │
│  id, subscription_id, date, message_count               │
└─────────────────────────────────────────────────────────┘
                     ▲
                     │
              Incrementé à
              chaque message
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                        PLAN                             │
│  id, name, price_cents, monthly_limit                   │
│  Free: 0€ / 50 msgs                                     │
│  Pro: 9.99€ / 2000 msgs                                │
│  Business: 29.99€ / illimité                           │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Instructions de Déploiement

### 1. Mettre à jour les variables d'environnement

```env
# LemonSqueezy (remplace Stripe)
LEMON_API_KEY=ls_xxx
LEMON_WEBHOOK_SECRET=whsec_xxx
LEMON_STORE_ID=store_xxx
```

### 2. Lancer les migrations

```bash
cd api
npm run prisma:migrate
npm run prisma:seed
```

### 3. Démarrer l'API

```bash
npm run build
npm start
```

### 4. Configurer les webhooks

```bash
# Telegram
curl -F "url=https://ton-domaine/telegram/webhook" \
  https://api.telegram.org/botTOKEN/setWebhook

# LemonSqueezy
# Via l'interface: Settings > Webhooks
# URL: https://ton-domaine/lemonsqueezy/webhook
# Secret: ton LEMON_WEBHOOK_SECRET
```

---

## ✅ Checklist de Fonctionnement

- [ ] Base de données migrée
- [ ] Plans créés (Free/Pro/Business)
- [ ] Variables LemonSqueezy configurées
- [ ] Webhook Telegram configuré
- [ ] Webhook LemonSqueezy configuré
- [ ] Token Telegram valide
- [ ] Clé OpenAI configurée

---

## 📈 Métriques à Surveiller

| Métrique | Description |
|----------|-------------|
| `subscription_active_count` | Nombre d'abonnés actifs |
| `mrr` | Revenue Mensuel Récurrent |
| `message_count` | Messages envoyés ce mois |
| `quota_limit_reached` | Users qui ont atteint leur limite |

---

## 🎯 Différenciateurs vs ClawNow

| | ClawNow | Ton SaaS |
|---|---------|----------|
| Prix | À la consommation | Abonnement fixe |
| Prévisibilité | Non | Oui |
| Limites | Crédits (confus) | Messages clairs |
| Marges | Variables | Garanties |

---

*Document généré par TechPartner Agent*
*11 février 2026*
