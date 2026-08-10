# Architecture de démarrage — Phase 1 (MVP)
## Plateforme de Lien de Paiement & Dashboard de Centralisation des Paiements Clients

**Date :** 10/08/2026
**Porteur du projet :** HY BUSINESS AND CO
**Référence :** [Cahier_des_Charges_Lien_de_Paiement_Dashboard.md](../../../Cahier_des_Charges_Lien_de_Paiement_Dashboard.md)
**Statut :** Validé pour rédaction du plan d'implémentation

---

## 1. Contexte

Ce document précise l'architecture technique de démarrage pour la Phase 1 (MVP) définie au §10 du cahier des charges :
création de liens de paiement, page de paiement hébergée, intégration PSP (sandbox), webhook de statut, dashboard
basique (liste + statuts + filtres), authentification admin.

**Hypothèse ouverte héritée du cahier des charges (§5.1, Phase 0)** : le support d'UnionPay par le PSP retenu doit
être confirmé par écrit avant la mise en production. Cette architecture est conçue pour rester interchangeable sur
ce point (voir §3 — couche d'abstraction PSP).

## 2. Choix validés avec le porteur de projet

| Décision | Choix retenu | Alternative envisagée |
|---|---|---|
| PSP de référence | PayDunya (via adapter) | CinetPay, dual CinetPay+international |
| Stack | Next.js + Node/Express + PostgreSQL | Next.js full-stack sans Express séparé |
| Structure du dépôt | Monorepo avec dossiers séparés `/web` et `/api` | Monorepo unifié, deux repos séparés |
| Emails transactionnels | Resend | SendGrid |
| Hébergement | Vercel (web) + Railway/Render (api + PostgreSQL) | Infrastructure imposée (non applicable ici) |

## 3. Vue d'ensemble de l'architecture

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────┐
│  /web (Next.js)  │◄───────►│   /api (Express)  │◄───────►│ PostgreSQL  │
│  - Dashboard admin│  REST   │  - Auth JWT       │         │ (Railway/   │
│  - Page paiement  │         │  - Liens/Transacs │         │  Render)    │
│  hébergée (client)│         │  - Webhooks PSP   │         └─────────────┘
│  Déployé: Vercel   │        │  Déployé: Railway │
└─────────────────┘         │  ou Render        │
                              └────────┬──────────┘
                                       │
                          ┌────────────┼────────────┐
                          ▼                          ▼
                  ┌───────────────┐         ┌────────────────┐
                  │ PayDunyaAdapter│         │  Resend (email) │
                  │ (interface PSP)│         │  admin + client │
                  └───────┬────────┘         └────────────────┘
                          ▼
                    PayDunya API
                 (checkout, webhook)
```

**Principe d'abstraction PSP** : toute la logique de paiement passe par une interface `PSPAdapter`
(`createPaymentSession`, `getTransactionStatus`, `parseWebhook`). Aucune logique métier ne dépend directement du SDK
PayDunya. Si le support UnionPay par PayDunya n'est pas confirmé lors de la Phase 0 (cadrage), un second adapter
(PSP international type Checkout.com/Adyen) peut être branché sans modifier le reste du système.

## 4. Modèle de données

Précise le §8 du cahier des charges avec les types et contraintes nécessaires à l'implémentation (Prisma / PostgreSQL) :

```prisma
model Client {
  id                String   @id @default(cuid())
  nom               String
  email             String?
  telephone         String?
  referenceDossier  String?
  liens             LienPaiement[]
  createdAt         DateTime @default(now())
}

model LienPaiement {
  id                    String    @id @default(cuid())
  reference             String    @unique          // ex. HYBC-2026-0001
  montant               Decimal?                    // null si montant libre
  montantLibre          Boolean   @default(false)
  devise                String    @default("XOF")
  description           String
  clientId              String
  client                Client    @relation(fields: [clientId], references: [id])
  modeAuthentification  String    // "3DS" | "STANDARD"
  statutLien            String    @default("actif") // actif | expire | desactive
  usageUnique           Boolean   @default(true)
  dateExpiration        DateTime?
  urlToken              String    @unique           // token opaque pour l'URL publique
  transactions          Transaction[]
  createdAt             DateTime  @default(now())
  createdBy             String                       // userId admin
}

model Transaction {
  id                    String   @id @default(cuid())
  lienId                String
  lien                  LienPaiement @relation(fields: [lienId], references: [id])
  idTransactionPSP      String?  @unique
  pspNom                String   // "paydunya"
  montant               Decimal
  devise                String
  reseauCarte           String?  // Visa | Mastercard | UnionPay
  modeAuthEffectif      String?  // 3DS réellement appliqué ou non
  statut                String   @default("en_attente") // en_attente|reussi|echoue|rembourse
  messageErreur         String?
  webhookPayload        Json?    // trace brute pour audit
  dateCreation          DateTime @default(now())
  dateMiseAJour         DateTime @updatedAt
}

model AdminUser {
  id                String   @id @default(cuid())
  nom               String
  email             String   @unique
  motDePasseHash    String
  role              String   @default("admin")
  derniereConnexion DateTime?
  createdAt         DateTime @default(now())
}

model AuditLog {
  id        String   @id @default(cuid())
  userId    String
  action    String   // "creation_lien" | "desactivation_lien" | "login" | ...
  cibleId   String?
  detail    Json?
  createdAt DateTime @default(now())
}
```

**Exigence sécurité (§6, §12 du cahier)** : aucun champ carte (numéro, CVV, expiration) n'existe dans ce schéma —
aucune donnée carte ne transite ni n'est stockée côté HY BUSINESS AND CO. Le formulaire carte est entièrement
délégué aux hosted fields / au checkout hébergé de PayDunya.

## 5. Flux de paiement (bout en bout)

1. **Admin crée un lien** — `POST /api/liens` génère `reference` + `urlToken` unique, enregistre en base avec
   statut `actif`.
2. **Client reçoit l'URL** (`/pay/:urlToken`) par email/SMS/WhatsApp (envoi manuel en Phase 1, hors périmètre
   technique de l'application).
3. **Client ouvre la page de paiement** (route publique Next.js) — l'API vérifie que le lien est actif et non
   expiré, puis affiche le récapitulatif de la transaction et le widget de checkout PayDunya (aucune saisie carte
   sur les serveurs de HY BUSINESS AND CO).
4. **Checkout PayDunya** — si `modeAuthentification = 3DS`, PayDunya déclenche l'authentification forte
   (OTP/biométrie banque émettrice) ; sinon le paiement s'exécute en mode standard.
5. **Webhook de notification** — `POST /api/webhooks/psp` reçoit la notification, `PSPAdapter.parseWebhook()`
   vérifie la signature, puis met à jour `Transaction.statut`. Si `usageUnique = true`, `LienPaiement.statutLien`
   passe à `expire`.
6. **Emails automatiques** (Resend) — l'admin est notifié (succès/échec) ; le client reçoit un reçu simple en cas
   de succès.
7. **Mise à jour du dashboard** — lecture directe depuis PostgreSQL, sans polling du PSP ; l'affichage se met à
   jour dès que le webhook est traité.

**Point de vigilance sécurité** : la page de retour affichée au client après paiement ne constitue jamais la source
de vérité du statut. Elle déclenche un simple `GET /api/transactions/:id/statut` pour affichage ; le statut
définitif provient toujours du webhook asynchrone signé, ce qui évite toute fraude par manipulation de l'URL de
retour.

## 6. Authentification & sécurité d'accès admin

- Session : JWT en cookie `httpOnly` + `secure`, expiration courte (~2h) avec refresh silencieux.
- Mot de passe : hashé avec `argon2`.
- 2FA/TOTP : reporté en Phase 2 conformément au plan de développement du cahier (§10) ; le modèle `AdminUser` et le
  middleware d'authentification sont conçus pour l'accueillir sans migration de schéma lourde.
- Chaque action sensible (création/désactivation de lien, connexion) est journalisée dans `AuditLog`
  (traçabilité exigée au §6 du cahier).

## 7. Gestion des erreurs et fiabilité des webhooks

- La signature du webhook PayDunya est vérifiée avant tout traitement ; un webhook non signé ou invalide est rejeté.
- Idempotence : `idTransactionPSP` est une contrainte unique, donc un webhook rejoué ne duplique pas la transaction.
- Si le traitement du webhook échoue (ex. base de données indisponible), l'API répond HTTP 500 pour déclencher le
  mécanisme de nouvelle tentative standard du PSP ; l'échec est journalisé pour audit manuel.
- `GET /api/liens/:id` renvoie HTTP 410 (Gone) si le lien est expiré ou désactivé, bloquant tout nouveau paiement —
  conforme au critère d'acceptation du §12 du cahier.

## 8. Structure de dossiers

```
lien-de-paiement-dashbord/
├── web/                        # Next.js — dashboard admin + page paiement publique
│   ├── app/
│   │   ├── (admin)/dashboard/  # protégé par middleware auth
│   │   ├── (admin)/liens/
│   │   ├── pay/[token]/        # page de paiement publique
│   │   └── login/
│   ├── components/
│   └── lib/api-client.ts       # appels vers /api
│
├── api/                        # Express — logique métier + intégration PSP
│   ├── src/
│   │   ├── routes/              # liens, transactions, webhooks, auth, export
│   │   ├── psp/
│   │   │   ├── PSPAdapter.ts        # interface commune
│   │   │   └── PayDunyaAdapter.ts   # implémentation PayDunya
│   │   ├── services/             # email, audit
│   │   ├── middleware/           # auth, errorHandler
│   │   └── db/                   # prisma schema + client
│   └── prisma/schema.prisma
│
└── docs/
```

## 9. Endpoints API (Phase 1)

Reprend le §9 du cahier ; le détail complet (payloads, codes de statut) sera précisé dans le plan d'implémentation.

| Méthode | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/login` | Connexion admin |
| `POST` | `/api/liens` | Créer un lien de paiement |
| `GET` | `/api/liens` | Liste des liens (filtres) |
| `GET` | `/api/liens/:id` | Détail d'un lien (410 si expiré/désactivé) |
| `PATCH` | `/api/liens/:id/desactiver` | Désactiver un lien actif |
| `GET` | `/api/transactions` | Liste des transactions (filtres, pagination) |
| `GET` | `/api/transactions/:id` | Détail d'une transaction |
| `POST` | `/api/webhooks/psp` | Réception des notifications PayDunya |

## 10. Déploiement

- **`web`** — Vercel (déploiement Git automatique, certificat SSL inclus).
- **`api` + PostgreSQL** — Railway ou Render (choix définitif selon le budget ; les deux offrent PostgreSQL managé,
  SSL et variables d'environnement chiffrées).
- Secrets (clés PayDunya, secret JWT, clé API Resend) exclusivement en variables d'environnement, jamais commitées.
- Sauvegarde quotidienne de PostgreSQL activée dès le déploiement initial (exigence §6 du cahier).

## 11. Tests (Phase 1)

- Tests unitaires sur `PSPAdapter` avec mocks des réponses PayDunya sandbox.
- Tests d'intégration sur les routes critiques : création de lien, réception de webhook (succès / échec / signature
  invalide), expiration de lien.
- Test manuel de bout en bout en environnement sandbox PayDunya avant mise en production (correspond à la Phase 0 —
  cadrage — du plan de développement du cahier).

## 12. Hors périmètre de cette architecture (rappel)

Conformément au §2 du cahier des charges, les éléments suivants ne sont pas couverts par cette architecture de
démarrage et seront traités en Phase 2/3 : paiements récurrents, Mobile Money, multi-utilisateurs avec rôles fins,
facturation PDF automatique, API publique, 2FA, QR code sur les liens, export CSV/Excel avancé, journal d'audit
consultable dans l'UI.

## 13. Risque ouvert à lever avant développement

Le support effectif d'UnionPay + 3‑D Secure par PayDunya n'est pas garanti par le cahier des charges (§5.1) et doit
être confirmé par écrit auprès de PayDunya avant de figer définitivement l'intégration (Phase 0 du plan de
développement). Cette architecture limite l'impact de ce risque grâce à la couche `PSPAdapter`, mais ne le résout
pas : c'est une action de cadrage, pas une action technique.
