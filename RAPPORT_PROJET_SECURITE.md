# 🛡️ DOSSIER TECHNIQUE & CAHIER DE SÉCURITÉ INFORMATIQUE
**Projet Universitaire : SecureAuth – Authentification, Contrôle d'Accès & Cryptographie**

---

## 📋 Informations du Projet
- **Nom de l'application :** SecureAuth
- **Domaine :** Sécurité des Applications Web & Cryptographie Appliquée
- **Technologies principales :** Python 3.11, Flask 3.0, SQLAlchemy, Bcrypt, Fernet (AES-256), HMAC-SHA256
- **Base de données :** SQLite (`instance/secure_app.db`)

---

## 1. 🎯 Présentation Générale et Objectifs

Le projet **SecureAuth** est une application web conçue selon les principes de la **défense en profondeur (Defense in Depth)**. Elle a pour but de fournir une démonstration concrète et opérationnelle des mécanismes de protection face aux cyberattaques réelles (Top 10 OWASP).

Le système sépare hermétiquement deux interfaces avec des niveaux de privilèges distincts :
1. **Interface Utilisateur (User) :**
   - Inscription et authentification avec validation de mot de passe stricte.
   - Gestion des informations personnelles et mise à jour du mot de passe.
   - Module de chiffrement symétrique de messages (AES-256).
   - Module de génération et de vérification d'empreintes d'intégrité (HMAC-SHA256).
2. **Interface Administrateur (Admin) :**
   - Tableau de bord avec indicateurs de sécurité en temps réel.
   - Gestion des comptes utilisateurs (activation, désactivation, promotion admin, suppression).
   - Journal d'audit exhaustif des événements de sécurité (Security Logs).

---

## 2. 🏛️ Architecture Technique & Modèles de Données

### A. Structure des Fichiers
```text
├── app/
│   ├── __init__.py          # Factory Pattern Flask, initialisation et en-têtes HTTP
│   ├── extensions.py        # Extensions : db, bcrypt, limiter, csrf
│   ├── models.py            # Modèles SQLAlchemy : User, SecureData, SecurityLog
│   ├── routes/
│   │   ├── auth.py          # Contrôleur d'authentification (/login, /register, /logout)
│   │   ├── user.py          # Contrôleur espace utilisateur (/dashboard, /encrypt, /hmac)
│   │   └── admin.py         # Contrôleur espace administrateur (/admin/users, /admin/logs)
│   └── utils/
│       ├── crypto.py        # Algorithmes AES-256, HMAC-SHA256, SHA-256
│       ├── security.py      # Décorateurs RBAC (@login_required, @admin_required), logs
│       └── helpers.py       # Validation et assainissement des entrées utilisateur
├── instance/
│   └── secure_app.db        # Base de données SQLite locale
├── templates/               # Vues HTML Jinja2 avec Dark Theme
├── static/                  # CSS (styles.css) et JavaScript (main.js)
├── config.py                # Gestion des environnements et politique CSP
├── run.py                   # Script de lancement du serveur
└── .env                     # Clés secrètes (AES_ENCRYPTION_KEY, HMAC_SECRET_KEY, etc.)
```

### B. Schéma de la Base de Données
1. **Table `users` :**
   - `id`, `email`, `password_hash` (hachage Bcrypt, jamais de texte clair), `nom`, `prenom`, `role` (`user` ou `admin`), `is_active` (booléen), `created_at`, `last_login`.
2. **Table `secure_data` :**
   - `id`, `user_id` (clé étrangère liée à `users`), `encrypted_content` (texte chiffré AES-256), `hmac_signature` (signature d'intégrité), `original_data_hash` (empreinte SHA-256), `created_at`.
3. **Table `security_logs` :**
   - `id`, `user_id` (optionnel), `action` (ex. connexion_réussie, accès_admin_refusé), `ip_address` (adresse IP cliente), `user_agent`, `success` (booléen), `details`, `timestamp`.

---

## 3. 🛡️ Les 6 Phases de Sécurité Détaillées

### 🔹 Phase 1 : Authentification & Sécurisation des Mots de Passe
- **Hachage adaptatif Bcrypt (Coût = 12) :**
  Contrairement à des fonctions rapides comme MD5 ou SHA-256 qui peuvent être attaquées par GPU à raison de milliards de tests par seconde, Bcrypt intègre un coût réglable (salage automatique + étirement de clé). Chaque hachage prend ~250ms, rendant les attaques par force brute irréalisables.
- **Politique stricte de mot de passe :**
  Validation obligatoire : Minimum 8 caractères, au moins 1 lettre majuscule, 1 chiffre, et 1 caractère spécial.
- **Résistance aux attaques temporelles (Timing Attacks) :**
  L'instruction `bcrypt.check_password_hash()` compare les empreintes en temps constant pour éviter toute déduction par mesure du temps d'exécution.
- **Message d'erreur neutre :**
  En cas d'échec de connexion, le serveur renvoie toujours *"Email ou mot de passe incorrect"* afin d'empêcher l'énumération des utilisateurs enregistrés.

### 🔹 Phase 2 : Sécurité des Sessions & Hygiène des Cookies
- **Attribut `HttpOnly = True` :**
  Le cookie de session est invisible pour le code JavaScript (`document.cookie`), bloquant le vol de session en cas de vulnérabilité XSS.
- **Attribut `SameSite = 'Lax'` :**
  Interdit l'envoi automatique du cookie de session lors d'une requête POST provenant d'un domaine externe.
- **Durée de vie contrôlée (TTL 30 minutes) :**
  La session expire automatiquement après 1800 secondes d'inactivité.
- **Destruction complète à la déconnexion :**
  L'appel à `/api/logout` vide entièrement la session (`session.clear()`) et révoque le cookie.

### 🔹 Phase 3 : Contrôle d'Accès Basé sur les Rôles (RBAC)
- **Décorateur `@login_required` :**
  Vérifie qu'un identifiant de session existe et interroge la base de données pour confirmer que l'utilisateur est toujours actif (`user.is_active == True`).
- **Décorateur `@admin_required` :**
  Contrôle le rôle de l'utilisateur (`user.role == 'admin'`). Si un utilisateur standard tente d'accéder à l'espace administration, l'accès est refusé (Code 403) et l'incident est consigné dans les journaux de sécurité.

### 🔹 Phase 4 : Cryptographie Symétrique (AES-256) & Intégrité (HMAC)
- **Chiffrement AES-256 (Module Fernet) :**
  Utilisation du standard Fernet : AES-128-CBC avec clé dérivée 256 bits, padding PKCS7 et vecteur d'initialisation (IV) aléatoire généré à chaque appel. Le même texte produit des représentations chiffrées totalement différentes à chaque fois.
- **Clés cryptographiques isolées :**
  Les clés `AES_ENCRYPTION_KEY` et `HMAC_SECRET_KEY` ne sont jamais écrites en dur dans le code ; elles sont injectées dynamiquement via le fichier d'environnement `.env`.
- **Signature d'intégrité HMAC-SHA256 :**
  Génération d'un code HMAC sur le texte chiffré.
- **Validation en temps constant :**
  Vérification de la signature à l'aide de `hmac.compare_digest()` afin de garantir que les données n'ont subi aucune altération malveillante.

### 🔹 Phase 5 : Défense en Profondeur (Réseau, Limites & En-têtes HTTP)
- **Limitation de débit (Rate Limiting avec Flask-Limiter) :**
  - Route `/api/login` : maximum 5 tentatives par tranche de 10 minutes par adresse IP.
  - Route `/api/register` : maximum 10 créations de compte par heure par adresse IP.
  - Neutralise les robots de force brute et les attaques par déni de service (DoS).
- **Protection Anti-CSRF (Flask-WTF & Intercepteur JS) :**
  - Un jeton CSRF est généré côté serveur et placé dans la balise `<meta name="csrf-token">`.
  - Le fichier `static/js/main.js` intercepte automatiquement chaque requête `fetch` et attache l'en-tête `X-CSRFToken`.
- **En-têtes de sécurité HTTP stricts :**
  - `Content-Security-Policy (CSP)` : Restreint l'exécution de scripts et le chargement de polices aux domaines explicitement approuvés.
  - `X-Frame-Options: DENY` : Empêche l'intégration du site dans une iframe (protection anti-clickjacking).
  - `X-Content-Type-Options: nosniff` : Empêche le MIME-sniffing par les navigateurs.
  - `X-XSS-Protection: 1; mode=block` : Active les filtres de protection XSS natifs des navigateurs.

### 🔹 Phase 6 : Traçabilité, Audit & Gestion des Incidents (SIEM/Audit Trail)
- **Table d'audit `security_logs` :**
  Chaque tentative de connexion (réussie ou échouée), modification de rôle, création de compte et accès non autorisé est enregistrée avec l'IP réelle, l'User-Agent et la date/heure UTC.
- **Zéro fuite d'informations sensibles :**
  Les fonctions d'audit ne consignent jamais de mots de passe, de clés cryptographiques ou de données en clair.
- **Console d'administration :**
  L'administrateur dispose d'une vue paginée et filtrable en temps réel pour analyser l'historique des connexions suspectes.

---

## 4. 📊 Matrice de Correspondance OWASP Top 10

| Risque OWASP Top 10 | Danger | Protection Implémentée dans le Projet |
|---|---|---|
| **A01: Broken Access Control** | Accès non autorisé aux pages admin | Décorateurs `@login_required`, `@admin_required`, vérification en base de données |
| **A02: Cryptographic Failures** | Vol de mots de passe ou de données sensibles | Hachage Bcrypt (coût 12), chiffrement AES-256 (Fernet), HMAC-SHA256, clés dans `.env` |
| **A03: Injection (SQL / XSS)** | Altération de la base ou injection de scripts | ORM SQLAlchemy (requêtes paramétrées), auto-échappement Jinja2, assainissement regex |
| **A04: Insecure Design** | Attaques par force brute | Rate Limiting (Flask-Limiter), politique stricte de complexité des mots de passe |
| **A05: Security Misconfiguration** | Fuite d'informations système | Pages d'erreurs 403/404/500 personnalisées, en-têtes HTTP de sécurité stricts (CSP, DENY) |
| **A07: Identification Failures** | Vol de cookies de session | Cookies `HttpOnly`, `SameSite=Lax`, expiration après 30 min, purge serveur à la déconnexion |
| **A09: Security Logging Failures** | Activités malveillantes indétectables | Journalisation centralisée dans `SecurityLog` avec IP, horodatage UTC et tableau d'audit |

---

## 5. 🧪 Scénarios de Démonstration & Soutenance

1. **Test Force Brute :**
   Entrer 6 fois un faux mot de passe sur la page `/login`. Le serveur bloque la requête avec une erreur `429 Too Many Requests`.
2. **Test Altération HMAC :**
   Sur `/encrypt`, chiffrer un message. Sur `/hmac`, tester le HMAC avec le message modifié d'une seule lettre : le résultat affiche `HMAC Invalide ✗`.
3. **Test Élévation de Privilège :**
   Se connecter avec un compte utilisateur normal, puis taper manuellement dans la barre d'adresse `http://127.0.0.1:5000/admin/dashboard`. L'accès est refusé et l'événement est archivé dans les logs d'audit.
4. **Vérification en Base de Données :**
   Ouvrir `instance/secure_app.db` : les mots de passe sont tous sous la forme `$2b$12$...` et les données enregistrées sont chiffrées en AES-256.

---

## 6. 📌 Identifiants de Démonstration

- **Compte Administrateur par défaut :**
  - **Email :** `admin@projet.com`
  - **Mot de passe :** `Admin123!`
- **Compte Utilisateur de test :**
  - Création libre depuis la page `/register`.
