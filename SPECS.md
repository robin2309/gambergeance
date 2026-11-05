# Ideas — Spécifications

Application web personnelle pour noter des idées de choses à **faire**, **investiguer**, **lire** et **regarder**, accessible depuis plusieurs appareils (ordinateur + téléphone).

---

## 1. Spécifications fonctionnelles

### 1.1 Utilisateur
- Un seul utilisateur (moi). Pas de multi-comptes, pas de partage.
- L'app est exposée sur internet : elle doit être protégée par un mot de passe.

### 1.2 Objet métier : Item
| Champ | Obligatoire | Description |
|---|---|---|
| Titre | Oui | Texte libre, 1 à 200 caractères |
| Lien | Non | URL `http://` ou `https://` |
| Catégorie | Oui | Une parmi : `À faire`, `À investiguer`, `À lire`, `À regarder` |
| Fait | — | Booléen, `false` à la création |
| Date de création | — | Automatique |

### 1.3 Fonctionnalités

**F1 — Ajouter un item**
- Formulaire en haut de page : titre, lien (optionnel), catégorie (sélecteur, mémorise la dernière catégorie utilisée).
- Validation : titre non vide ; si un lien est saisi, il doit être une URL http(s) valide.
- Touche Entrée = valider. Après ajout, le formulaire se vide et le focus revient sur le titre.
- Ajout rapide : si l'utilisateur colle une URL seule dans le champ titre, elle est déplacée dans le champ lien (le titre reste à saisir).

**F2 — Consulter les items par catégorie**
- La page affiche une section par catégorie, dans l'ordre : À faire, À investiguer, À lire, À regarder.
- Chaque section affiche son nombre d'items non faits.
- Dans une section : items non faits d'abord (plus récent en haut), puis items faits (grisés, barrés), repliés par défaut derrière « Afficher les X terminés ».
- Une section vide affiche un message discret (« Rien pour l'instant »).
- Si un item a un lien : le titre est cliquable et ouvre le lien dans un nouvel onglet ; le domaine (ex. `youtube.com`) est affiché à côté en petit.

**F3 — Marquer comme fait / non fait**
- Case à cocher sur chaque item. Changement immédiat (UI optimiste).

**F4 — Modifier un item**
- Bouton « Modifier » : édition en place du titre, du lien et de la catégorie. Enregistrer / Annuler (Échap).

**F5 — Supprimer un item**
- Bouton « Supprimer » avec confirmation.

**F6 — Filtrer**
- Onglets/chips en haut de liste : « Tout » + une par catégorie, pour n'afficher qu'une section (utile sur mobile).
- Champ de recherche texte (filtre côté client sur titre et lien).

**F7 — Authentification**
- Accès protégé par un mot de passe unique défini côté serveur.
- Une fois connecté sur un appareil, on reste connecté (session longue, 1 an).
- Bouton « Se déconnecter ».

### 1.4 Ergonomie
- Mobile-first : utilisable confortablement sur un écran de 375 px de large, cibles tactiles ≥ 44 px.
- Thème clair/sombre automatique (`prefers-color-scheme`).
- Interface en français.
- Installable sur l'écran d'accueil du téléphone (manifest PWA minimal : nom, icône, `display: standalone`). Pas de mode hors-ligne requis.

### 1.5 Hors périmètre (v1)
- Multi-utilisateurs, partage.
- Catégories personnalisées, tags, priorités, dates d'échéance.
- Mode hors-ligne / synchronisation.
- Récupération automatique du titre d'une page à partir de son URL.

---

## 2. Spécifications techniques

### 2.1 Stack
| Couche | Choix | Raison |
|---|---|---|
| Runtime | Node.js ≥ 20 (ESM, JavaScript, pas de TypeScript ni d'étape de build) | Simplicité |
| Serveur HTTP | [Hono](https://hono.dev) + `@hono/node-server` | Léger, API simple |
| Base de données | SQLite via `better-sqlite3` | Un seul fichier, zéro administration |
| Front | Une page `public/index.html` + `public/app.js` + `public/style.css`, JS vanilla | Pas de build, servie par le même serveur |
| Hébergement | Fly.io, 1 machine, volume persistant pour le fichier SQLite | Accès multi-appareils, HTTPS fourni |

Dépendances déjà installées : `hono`, `@hono/node-server`, `better-sqlite3`.

### 2.2 Arborescence
```
ideas/
├── server.js          # app Hono : auth, routes API, fichiers statiques
├── db.js              # connexion SQLite, migration, requêtes
├── public/
│   ├── index.html
│   ├── login.html
│   ├── app.js
│   ├── style.css
│   ├── manifest.webmanifest
│   └── icon.svg
├── Dockerfile
├── fly.toml
├── .env.example
└── package.json
```

### 2.3 Configuration (variables d'environnement)
| Variable | Défaut | Rôle |
|---|---|---|
| `PORT` | `3000` | Port d'écoute |
| `DB_PATH` | `./data/ideas.db` | Chemin du fichier SQLite (`/data/ideas.db` en prod) |
| `APP_PASSWORD` | — (obligatoire) | Mot de passe d'accès. Le serveur refuse de démarrer s'il est absent |
| `SESSION_SECRET` | — (obligatoire) | Secret de signature du cookie de session |

### 2.4 Modèle de données
```sql
CREATE TABLE IF NOT EXISTS items (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  title       TEXT    NOT NULL CHECK (length(title) BETWEEN 1 AND 200),
  url         TEXT,
  category    TEXT    NOT NULL CHECK (category IN ('todo','investigate','read','watch')),
  done        INTEGER NOT NULL DEFAULT 0 CHECK (done IN (0,1)),
  created_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  updated_at  TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);
CREATE INDEX IF NOT EXISTS idx_items_category ON items(category, done, created_at);
```
- Créer le dossier de `DB_PATH` s'il n'existe pas ; activer `PRAGMA journal_mode = WAL`.
- Libellés affichés : `todo` → À faire, `investigate` → À investiguer, `read` → À lire, `watch` → À regarder. La liste des catégories est définie à un seul endroit côté serveur et exposée au front (voir `GET /api/categories`).

### 2.5 API REST
Toutes les routes `/api/*` exigent une session valide (sinon `401`). Corps et réponses en JSON.

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| `GET` | `/api/categories` | — | `200 [{ "key": "todo", "label": "À faire" }, …]` dans l'ordre d'affichage |
| `GET` | `/api/items` | — | `200 [Item]` triés par catégorie (ordre ci-dessus), puis `done` ASC, puis `created_at` DESC |
| `POST` | `/api/items` | `{ title, url?, category }` | `201 Item` |
| `PATCH` | `/api/items/:id` | sous-ensemble de `{ title, url, category, done }` | `200 Item` ; met à jour `updated_at` |
| `DELETE` | `/api/items/:id` | — | `204` |

Format `Item` :
```json
{ "id": 1, "title": "…", "url": "https://…" , "category": "read", "done": false,
  "createdAt": "2026-11-05T10:00:00.000Z", "updatedAt": "…" }
```
`url` vaut `null` si absent ; `done` est un booléen JSON.

**Validation serveur** (ne jamais faire confiance au front) :
- `title` : chaîne, trim, 1–200 caractères.
- `url` : chaîne vide → `null` ; sinon `new URL()` doit réussir et le protocole être `http:` ou `https:` (refuser `javascript:` etc.).
- `category` : une des clés connues.
- `done` : booléen.
- Erreur de validation → `400 { "error": "message lisible" }`. Id inexistant → `404`.

### 2.6 Authentification
- `GET /login` sert `login.html` (un champ mot de passe).
- `POST /login` (form) : compare le mot de passe à `APP_PASSWORD` en temps constant (`crypto.timingSafeEqual` sur des hash SHA-256). Succès → cookie de session signé (Hono `setSignedCookie` avec `SESSION_SECRET`), `HttpOnly`, `Secure` en prod, `SameSite=Lax`, `Max-Age` 1 an, puis redirection vers `/`. Échec → redirection vers `/login?error=1`.
- `POST /logout` : supprime le cookie, redirige vers `/login`.
- Middleware : toute requête hors `/login`, `/manifest.webmanifest` et `/icon.svg` sans session valide → `302 /login` pour les pages, `401` pour `/api/*`.
- Limitation simple des tentatives : délai de 1 s sur chaque échec de login.

### 2.7 Front
- `app.js` charge `/api/categories` et `/api/items` au démarrage, garde l'état en mémoire et re-rend la liste après chaque action.
- Mises à jour optimistes (cocher, supprimer) avec retour arrière + message d'erreur si l'API échoue.
- Une réponse `401` redirige vers `/login`.
- Construction du DOM via `textContent` / `createElement` (pas d'`innerHTML` avec des données utilisateur) pour éviter les XSS. Liens ouverts avec `target="_blank" rel="noopener noreferrer"`.
- CSS : variables de couleurs sur `:root`, redéfinies dans `@media (prefers-color-scheme: dark)`. Pas de framework CSS.

### 2.8 Déploiement (Fly.io)
- `Dockerfile` : image `node:22-slim`, `npm ci --omit=dev`, `CMD ["node","server.js"]`.
- `fly.toml` : `internal_port = 3000`, `force_https = true`, montage d'un volume `ideas_data` sur `/data`, `DB_PATH=/data/ideas.db`, 1 machine (SQLite n'accepte qu'une instance), `auto_stop_machines = "stop"` et `min_machines_running = 0` pour limiter le coût.
- Secrets posés via `fly secrets set APP_PASSWORD=… SESSION_SECRET=…`.
- Documenter dans un `README.md` : lancement local (`npm run dev` avec `.env`), premier déploiement (`fly launch --no-deploy`, `fly volumes create ideas_data --size 1`, `fly secrets set …`, `fly deploy`), sauvegarde de la base (`fly ssh sftp get /data/ideas.db`).

### 2.9 Critères d'acceptation
1. Sans mot de passe, `/` redirige vers `/login` et `/api/items` répond `401`.
2. Après connexion, ajouter « Dune » sans lien dans « À lire » : l'item apparaît en tête de la section, compteur à jour.
3. Ajouter un item avec le lien `https://www.youtube.com/watch?v=x` : titre cliquable, `youtube.com` affiché.
4. Un lien `javascript:alert(1)` ou `ftp://x` est refusé (`400`) avec message affiché.
5. Cocher un item le passe dans les « terminés » repliés ; le décocher le remet en haut.
6. Modifier la catégorie d'un item le déplace dans la bonne section.
7. Supprimer demande confirmation puis retire l'item.
8. Après redémarrage du serveur (ou redeploy Fly), les items sont toujours là.
9. Un item ajouté depuis le téléphone apparaît sur l'ordinateur après rechargement.
10. Affichage correct et sans scroll horizontal à 375 px de large, en clair et en sombre.
