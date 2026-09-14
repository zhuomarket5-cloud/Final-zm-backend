# ZhuoMarket Backend

Backend REST production-ready pour la marketplace mobile **ZhuoMarket** : Node.js + Express + PostgreSQL (`pg`) + JWT/bcryptjs + Cloudinary.

## Important: compatibilité Cloudinary

Ce projet **n'utilise volontairement pas `multer-storage-cloudinary`**. Le package `multer-storage-cloudinary` est actuellement en version 4.0.0 et très ancien, tandis que le SDK officiel `cloudinary` est en 2.11.0. Pour éviter les conflits de peer-dependencies `ERESOLVE`, le projet utilise `multer.memoryStorage()` puis l'API `cloudinary.uploader.upload_stream()` directement. Cela reste un upload Cloudinary réel et évite une dépendance de stockage non nécessaire.

## Arborescence

```text
zhuomarket-backend/
├── server.js
├── package.json
├── .env.example
├── .gitignore
├── README.md
├── config/
│   ├── env.js
│   ├── db.js
│   └── cloudinary.js
├── middleware/
│   ├── auth.js
│   └── upload.js
├── migrations/
│   ├── 001_initial.js
│   └── 002_non_destructive.js
├── routes/
│   ├── index.js
│   ├── auth.js
│   ├── products.js
│   ├── promotions.js
│   ├── orders.js
│   ├── trades.js
│   ├── notifications.js
│   ├── messages.js
│   ├── admin.js
│   ├── users.js
│   ├── paymentMethods.js
│   ├── streaming.js
│   ├── uploads.js
│   ├── chatbot.js
│   └── brands.js
├── services/
│   ├── auth.js
│   ├── cloudinary.js
│   ├── notifications.js
│   └── openai.js
└── utils/
    ├── http.js
    └── normalize.js
```

## Variables Render

Dans Render > Web Service > Environment, ajoute toutes les variables de `.env.example`.

Minimum obligatoire : `DATABASE_URL`, `JWT_SECRET`, `OWNER_EMAIL`, `OWNER_PASSWORD`, `OWNER_NAME`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`.

`OPENAI_API_KEY` est optionnelle. Sans elle, `/api/chatbot` ne simule aucune réponse IA et indique que l'IA est désactivée.

## PostgreSQL

Crée une base PostgreSQL Render et copie sa `Internal Database URL` dans `DATABASE_URL` si le backend et la base sont dans Render. Les migrations sont non destructives : elles utilisent `CREATE TABLE IF NOT EXISTS` et `ALTER TABLE IF NOT EXISTS`.

## Déploiement Render

1. Mets ces fichiers dans un dépôt GitHub.
2. Render > New > Web Service > connecte le dépôt.
3. Runtime : Node.
4. Build Command : `npm install`.
5. Start Command : `npm start`.
6. Ajoute les variables d'environnement.
7. Déploie.
8. Vérifie `https://TON-SERVICE.onrender.com/health`.
9. Une réponse HTTP 200 doit contenir `ok: true` et `database: "ok"`.

## Owner protégé

Au démarrage, le backend crée le compte défini par `OWNER_EMAIL`, `OWNER_PASSWORD`, `OWNER_NAME` s'il n'existe pas. Si l'adresse existe déjà, elle est remise au rôle `owner`. Un owner ne peut pas être rétrogradé via les routes d'administration normales.

## Auth

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `GET /api/auth/me`
- `PATCH /api/auth/me`
- `POST /api/auth/me/avatar` avec multipart `image`
- `POST /api/auth/logout`

## Catalogue / produits

- `GET /api/products`
- `GET /api/products/:id`
- Admin : `POST /api/products`
- Admin : `PUT /api/products/:id`
- Admin : `PATCH /api/products/:id`
- Admin : `DELETE /api/products/:id`
- `GET /api/brands`

Les uploads produit utilisent Cloudinary réel. Le backend accepte `images` et/ou des fichiers multipart `images`.

## Promotions / publicité

- `GET /api/promotions`
- Admin : `GET /api/promotions/admin`
- Admin : `POST /api/promotions`
- Admin : `PUT /api/promotions/:id`
- Admin : `PATCH /api/promotions/:id` avec `{ "status": "published" | "paused" }`
- Admin : `DELETE /api/promotions/:id`

## Règle critique du stock

**La création d'une commande ne décrémente jamais le stock.**

`POST /api/orders` calcule le total depuis les prix PostgreSQL et crée la commande avec `status=pending`.

La décrémentation se produit uniquement via :

`POST /api/orders/:id/validate`

Cette route exécute une transaction PostgreSQL : `BEGIN` → verrouillage de la commande et de chaque produit avec `SELECT ... FOR UPDATE` → contrôle du stock → décrémentation → validation → `COMMIT`. Une seconde validation de la même commande est refusée et un stock insuffisant est refusé avant toute décrémentation. Cela empêche les stocks négatifs lors de validations concurrentes.

## Commandes

- Client : `POST /api/orders`
- Client/admin : `GET /api/orders`
- Admin : `GET /api/orders?admin=true`
- Admin : `POST /api/orders/:id/validate`
- Admin : `PATCH /api/orders/:id/status`
- Admin : `PATCH /api/orders/:id/payment`
- Client : `PATCH /api/orders/:id/payment-confirmation`

Les méthodes manuelles prévues sont `moncash`, `natcash`, `zelle`, `cod`, `card`, `paypal`. Aucun paiement n'est simulé. Le champ `payment_method` est persisté. Les intégrations de paiement externes ne sont pas activées par défaut.

## Trade

- Client : `POST /api/trades`
- Client/admin : `GET /api/trades`
- Admin : `PATCH /api/trades/:id` avec `pending`, `accepted` ou `rejected`

## Support

Client : `GET/POST /api/messages`.

Admin : `GET /api/messages/admin`, `GET /api/messages/admin/:id`, `POST /api/messages/admin/:id`, `GET /api/messages/unread`.

L'IA OpenAI est optionnelle. Elle ne s'active que si `OPENAI_API_KEY` est fournie. Aucun faux message IA n'est généré si la clé est absente.

## Notifications

- `GET /api/notifications`
- `GET /api/notifications/due-reminders`
- `PATCH /api/notifications/:id/read`
- `POST /api/notifications/mark-all-read`
- `GET/PATCH /api/notifications/preferences`

## Streaming

- `GET /api/streaming-plans`
- `POST /api/streaming-orders`
- Admin : `PUT /api/admin/streaming-plans`
- Admin : `GET /api/admin/streaming-orders`

Netflix et Disney+ sont persistés en PostgreSQL et leurs prix peuvent être configurés par l'admin.

## Paiements manuels

- `GET /api/payment-methods` public, uniquement méthodes activées.
- Admin : `GET/POST /api/admin/payment-methods`
- Admin : `PATCH /api/admin/payment-methods/:id`

Les instructions de paiement (numéro, compte, email, handle, etc.) restent côté backend et sont renvoyées au frontend selon la méthode activée.

## Uploads génériques

Admin : `POST /api/uploads` avec multipart `images`.

## Test local

```bash
cp .env.example .env
# remplis .env
npm install
npm run check
npm start
```

Puis :

```bash
curl http://localhost:10000/health
```

## Frontend ZhuoMarket fourni

Le frontend fourni dans la conversation consomme notamment `/api/auth/*`, `/api/products`, `/api/promotions`, `/api/orders`, `/api/users`, `/api/admin/stats`, `/api/notifications`, `/api/trades`, `/api/messages`, `/api/uploads`, `/api/payment-methods`, `/api/admin/payment-methods`, `/api/streaming-plans`, `/api/streaming-orders` et `/api/chatbot`. Le backend est structuré pour ces routes sans exiger de modification de l'UI.


## GitHub mobile distribution
This version intentionally keeps the backend in a single `server.js` so GitHub mobile upload does not need folder creation. Internal config/routes/services/migrations are embedded in that file.
