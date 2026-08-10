# Cahier des Charges
## Plateforme de Lien de Paiement & Dashboard de Centralisation des Paiements Clients

**Version :** 1.0
**Date :** 10/08/2026
**Porteur du projet :** HY BUSINESS AND CO

---

## ⚠️ Hypothèses à valider avant développement

Deux points du brief initial nécessitent une confirmation avant de lancer le développement avec Claude Code. Le document ci-dessous a été rédigé sur la base des hypothèses suivantes — à corriger si elles sont fausses :

1. **« 3DS ou 2DS »** : il n'existe pas de norme officielle « 2DS ». On considère ici que cela désigne un paiement **sans authentification forte 3‑D Secure** (saisie simple des données carte : numéro, date d'expiration, CVV), par opposition au **3‑D Secure** (avec code OTP/SMS ou biométrie envoyé par la banque émettrice). Le document utilise donc les termes **« Paiement 3DS »** et **« Paiement standard (non-3DS) »**.
2. **« L'utilisateur pourra choisir »** : on suppose qu'il s'agit de **l'administrateur du cabinet qui crée le lien de paiement** (choix fait en amont, au moment de la génération du lien), et non du client payeur final qui choisirait lui-même sur la page de paiement. C'est l'interprétation la plus cohérente d'un point de vue risque/conformité (voir §6.2). À confirmer.

---

## 1. Contexte et objectifs

HY BUSINESS AND CO réalise pour ses clients des prestations de conseil, d'intermédiation bancaire/financière et de recherche de financement. Les encaissements clients par carte bancaire sont aujourd'hui dispersés (virements, dépôts directs, etc.), sans outil unique de suivi.

**Objectif du projet :** développer une plateforme permettant de :
- Générer des **liens de paiement** (à usage unique ou réutilisables) à envoyer aux clients par email/WhatsApp/SMS ;
- Accepter les paiements par carte **Visa, Mastercard et UnionPay** ;
- Choisir, à la création du lien, si le débit s'effectue avec **authentification 3‑D Secure** ou en **paiement standard** ;
- Centraliser l'ensemble des paiements clients dans un **dashboard unique** (suivi, statuts, réconciliation, reporting).

---

## 2. Périmètre du projet

### Inclus (MVP)
- Génération de liens de paiement paramétrables (montant, devise, description, client, date d'expiration)
- Page de paiement hébergée (hosted payment page) sécurisée
- Support Visa / Mastercard / UnionPay via un prestataire de paiement (PSP)
- Choix 3DS / non-3DS à la création du lien
- Dashboard : liste des liens, statuts des transactions, filtres, recherche, export
- Authentification administrateur (email + mot de passe, idéalement 2FA)
- Notifications de statut (email a minima)

### Exclu du MVP (V2 potentielle)
- Paiements récurrents / abonnements
- Mobile Money (Orange Money, MTN MoMo, Wave) — à évaluer séparément
- Multi-utilisateurs avec rôles fins (comptable, commercial, admin)
- Facturation automatique (génération de reçus PDF)
- API publique pour intégration à d'autres outils internes du cabinet

---

## 3. Acteurs et rôles

| Acteur | Description |
|---|---|
| **Administrateur (HY BUSINESS AND CO)** | Crée les liens de paiement, consulte le dashboard, exporte les données |
| **Client payeur** | Reçoit le lien, saisit ses informations de carte, effectue le paiement |
| **PSP (prestataire de paiement)** | Traite la transaction, gère la conformité PCI DSS, déclenche le 3DS le cas échéant |
| **Banque émettrice** | Valide l'authentification 3DS côté porteur de carte |

---

## 4. Spécifications fonctionnelles

### 4.1 Création d'un lien de paiement
L'administrateur doit pouvoir renseigner :
- Montant (fixe, ou libre si le client doit choisir le montant)
- Devise (XOF par défaut ; prévoir extensibilité EUR/USD)
- Description / motif du paiement
- Nom et contact du client (email, téléphone) — pour rattachement dans le dashboard
- Dossier ou référence interne associée (ex. numéro de dossier HY BUSINESS AND CO)
- Mode d'authentification : **3DS** ou **Standard**
- Date d'expiration du lien (optionnelle)
- Lien à usage unique ou réutilisable

Le système génère alors une **URL unique** et, idéalement, un **QR code**.

### 4.2 Page de paiement (côté client)
- Page hébergée, responsive (mobile/desktop), aux couleurs du cabinet
- Récapitulatif de la transaction (montant, motif, bénéficiaire)
- Formulaire carte (numéro, expiration, CVV) via les champs sécurisés du PSP (« hosted fields » ou iframe) — **aucune donnée carte ne doit transiter par les serveurs de HY BUSINESS AND CO**
- Détection automatique du réseau de carte (Visa / Mastercard / UnionPay) à la saisie
- Déclenchement du parcours 3DS si ce mode a été sélectionné à la création du lien
- Page de confirmation (succès / échec) avec message clair

### 4.3 Dashboard de centralisation
- Tableau des transactions avec colonnes : date, client, montant, devise, mode (3DS/standard), réseau de carte, statut (en attente / réussi / échoué / expiré / remboursé), référence
- Filtres : période, statut, client, mode d'authentification, réseau de carte
- Recherche par nom de client ou référence
- Vue détail d'une transaction (historique des événements, ID PSP, éventuel motif d'échec)
- Export CSV / Excel des transactions sur une période donnée
- Indicateurs synthétiques (KPI) : total encaissé sur la période, taux de succès des paiements, taux d'abandon, répartition par réseau de carte
- Gestion des liens actifs (désactivation manuelle possible)

### 4.4 Notifications
- Email automatique à l'administrateur à chaque paiement réussi/échoué
- Email de confirmation au client payeur après paiement réussi (reçu simple)
- (V2) Webhook sortant pour intégration future avec d'autres outils du cabinet

### 4.5 Authentification & sécurité d'accès au dashboard
- Connexion admin par email + mot de passe (hashé, ex. bcrypt/argon2)
- Recommandé : authentification à deux facteurs (2FA/TOTP)
- Session avec expiration automatique
- Journal des connexions et actions sensibles (audit log)

---

## 5. Réseaux de cartes et choix du prestataire de paiement (PSP)

### 5.1 Point d'attention important — UnionPay
La majorité des agrégateurs de paiement actifs en Côte d'Ivoire (CinetPay, PayDunya, FedaPay) mettent en avant le support **Visa et Mastercard** ainsi que le Mobile Money local ; **le support explicite d'UnionPay n'est pas garanti par tous ces acteurs et doit être vérifié directement auprès du PSP retenu** avant tout développement. Si UnionPay est une exigence non négociable, il faudra probablement combiner :
- un agrégateur local (CinetPay/PayDunya/FedaPay) pour Visa/Mastercard/XOF, **et/ou**
- un PSP international acceptant UnionPay (ex. offres cartes internationales de type Checkout.com, Adyen, ou solutions bancaires locales avec passerelle 3DS comme celles proposées par les banques commerciales de la place).

**Recommandation :** prévoir une phase de cadrage courte (1 à 2 jours) avec 2-3 PSP candidats pour confirmer, par écrit, le support effectif d'UnionPay + 3DS avant de figer l'architecture d'intégration.

### 5.2 Exigences vis-à-vis du PSP
Le PSP retenu doit fournir :
- Acceptation Visa, Mastercard, UnionPay
- Support du 3‑D Secure 2.x et possibilité de transactions sans 3DS (selon les règles du PSP et du schéma de carte)
- Conformité **PCI DSS** (idéalement SAQ A via hosted fields / iframe, pour éviter que HY BUSINESS AND CO ne manipule des données carte)
- API de création de session de paiement + Webhooks de notification de statut
- Environnement de test (sandbox) avant mise en production
- Devise XOF supportée nativement (ou conversion transparente)

### 5.3 Conformité réglementaire
- Respect des exigences de la **BCEAO/UEMOA** applicables aux prestataires de services de paiement et à leurs partenaires
- Le PSP doit être agréé/habilité dans la zone UEMOA pour l'activité concernée
- HY BUSINESS AND CO conserve un rôle de donneur d'ordre technique ; vérifier si le montage nécessite un statut particulier (agent, apporteur d'affaires) selon la réglementation en vigueur — **point à valider avec un conseil juridique/réglementaire**, hors périmètre technique de ce cahier des charges.

---

## 6. Exigences non-fonctionnelles

| Catégorie | Exigence |
|---|---|
| **Sécurité** | Aucune donnée de carte stockée côté HY BUSINESS AND CO ; TLS 1.2+ sur toutes les pages ; hosted fields du PSP |
| **Conformité** | PCI DSS (niveau du PSP), respect BCEAO/UEMOA |
| **Disponibilité** | Page de paiement disponible 99,5 %+ ; monitoring des échecs de webhook |
| **Performance** | Chargement de la page de paiement < 2 secondes |
| **Traçabilité** | Historique complet et horodaté de chaque transaction et de chaque action admin |
| **Localisation** | Interface en français, montants en XOF (format local) |
| **Sauvegarde** | Sauvegarde quotidienne de la base de données (transactions, liens, utilisateurs) |

---

## 7. Architecture technique proposée (indicative)

> À adapter par Claude Code selon les contraintes réelles du PSP choisi.

- **Frontend Dashboard :** application web (ex. React/Next.js) — authentification admin, tableaux, filtres, export
- **Page de paiement :** générée côté serveur ou via widget/iframe fourni par le PSP
- **Backend / API :** service applicatif (ex. Node.js/Express ou équivalent) exposant :
  - création de lien de paiement
  - récupération du statut d'une transaction
  - réception des webhooks du PSP (confirmation asynchrone de paiement)
- **Base de données :** relationnelle (PostgreSQL recommandé) pour la fiabilité transactionnelle
- **Intégration PSP :** SDK/API du PSP retenu, clés API stockées en variables d'environnement sécurisées (jamais en dur dans le code)
- **Hébergement :** à définir (cloud avec certificat SSL, sauvegardes automatiques)

---

## 8. Modèle de données (entités principales)

**Lien de paiement**
`id, référence, montant, devise, description, client_id, mode_authentification (3DS/standard), statut_lien (actif/expiré/désactivé), date_creation, date_expiration, usage_unique (bool)`

**Transaction**
`id, lien_id, id_transaction_PSP, montant, devise, réseau_carte (Visa/Mastercard/UnionPay), mode_authentification_effectif, statut (en_attente/réussi/échoué/remboursé), date, message_erreur`

**Client**
`id, nom, email, téléphone, référence_dossier_HYBC`

**Utilisateur admin**
`id, nom, email, mot_de_passe_hash, rôle, date_dernière_connexion`

---

## 9. Spécification API (endpoints indicatifs)

| Méthode | Endpoint | Description |
|---|---|---|
| `POST` | `/api/liens` | Créer un nouveau lien de paiement |
| `GET` | `/api/liens/:id` | Détail d'un lien |
| `GET` | `/api/liens` | Liste des liens (avec filtres) |
| `PATCH` | `/api/liens/:id/desactiver` | Désactiver un lien actif |
| `GET` | `/api/transactions` | Liste des transactions (filtres, pagination) |
| `GET` | `/api/transactions/:id` | Détail d'une transaction |
| `POST` | `/api/webhooks/psp` | Réception des notifications du PSP |
| `GET` | `/api/export` | Export CSV/Excel des transactions |
| `POST` | `/api/auth/login` | Connexion admin |

---

## 10. Plan de développement suggéré (par phases)

**Phase 0 — Cadrage (avant dev)**
- Choix définitif du PSP (confirmation UnionPay + 3DS)
- Ouverture du compte marchand PSP + accès sandbox

**Phase 1 — MVP**
- Création de lien de paiement (formulaire admin simple)
- Page de paiement + intégration PSP (sandbox)
- Webhook de statut + mise à jour transaction
- Dashboard basique (liste + statuts + filtres)
- Authentification admin

**Phase 2 — Renforcement**
- Export CSV/Excel, KPI, recherche avancée
- 2FA, journal d'audit
- Notifications email automatisées
- QR code sur les liens

**Phase 3 — Extensions (optionnel)**
- Mobile Money
- Paiements récurrents
- Multi-utilisateurs avec rôles

---

## 11. Livrables attendus

- Application dashboard fonctionnelle (accès web sécurisé)
- Système de génération de liens de paiement opérationnel en production
- Documentation technique (variables d'environnement, déploiement, gestion des clés API)
- Guide d'utilisation rapide pour les administrateurs HY BUSINESS AND CO

---

## 12. Critères d'acceptation

- [ ] Un lien de paiement créé avec mode « Standard » aboutit à un paiement sans étape 3DS
- [ ] Un lien de paiement créé avec mode « 3DS » déclenche systématiquement l'authentification forte
- [ ] Les trois réseaux (Visa, Mastercard, UnionPay) sont acceptés et correctement identifiés dans le dashboard
- [ ] Aucune donnée de carte brute n'est visible ni stockée côté serveur HY BUSINESS AND CO
- [ ] Le statut d'une transaction se met à jour automatiquement via webhook, sans action manuelle
- [ ] L'export du dashboard reflète fidèlement les transactions sur la période sélectionnée
- [ ] Un lien expiré ou désactivé n'est plus utilisable pour un paiement
