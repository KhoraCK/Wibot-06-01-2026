# WIBOT Backend - Documentation Technique

## Vue d'ensemble

Le backend WIBOT est compose de 3 services Docker orchestres par Docker Compose :
- **PostgreSQL** : Base de donnees
- **n8n** : Moteur de workflows / API endpoints
- **Nginx** : Reverse proxy + serveur frontend

L'intelligence artificielle est fournie par **Mistral AI** via les nodes natifs n8n.

---

## Architecture

```
                    ┌─────────────────┐
                    │   Client Web    │
                    │  (React SPA)    │
                    └────────┬────────┘
                             │ HTTP :8080
                    ┌────────▼────────┐
                    │     Nginx       │
                    │ (Reverse Proxy) │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    /webhook/*        Static files      /health
           │          (React build)
           │
    ┌──────▼──────┐
    │     n8n     │
    │ (Workflows) │
    └──────┬──────┘
           │
    ┌──────▼──────┐     ┌─────────────┐
    │ PostgreSQL  │     │ Mistral API │
    │   (Data)    │     │  (Cloud)    │
    └─────────────┘     └─────────────┘
```

---

## Structure des fichiers

```
wibot-backend/
├── docker-compose.yml        # Orchestration des 3 services
├── .env                      # Variables d'environnement
├── .env.example              # Template des variables
├── nginx.conf                # Configuration Nginx
├── init.sql                  # Script init PostgreSQL
├── frontend/                 # Build React (servi par Nginx)
│   ├── index.html
│   └── assets/
└── workflows/                # Workflows n8n (JSON)
    ├── auth_login.json
    ├── chat_main.json
    ├── get_conversations.json
    ├── get_messages.json
    └── create_conversation.json
```

---

## Services Docker

### PostgreSQL

| Parametre | Valeur |
|-----------|--------|
| Image | `postgres:14-alpine` |
| Container | `wibot-postgres` |
| Port expose | `5433:5432` |
| Database | `wibot` |
| User | `widip` |
| Password | `widip_secure_password_2024` |

**Volumes :**
- `postgres_data` : Donnees persistantes
- `./init.sql` : Script d'initialisation

**Tables creees :**
- `users` : Utilisateurs WIBOT
- `conversations` : Conversations chat
- `messages` : Messages des conversations
- `user_token_usage` : Compteur tokens mensuel
- `n8n_chat_histories` : Historique chat (cree par n8n)

---

### n8n

| Parametre | Valeur |
|-----------|--------|
| Image | `n8nio/n8n:latest` |
| Container | `wibot-n8n` |
| Port expose | `5679:5678` |
| Auth | `admin` / `wibot_admin_2024` |
| Timezone | `Europe/Paris` |

**Variables d'environnement :**
```env
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=wibot_admin_2024
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_DATABASE=wibot
JWT_SECRET=wibot_jwt_secret_key_minimum_32_chars_2024
```

**Volumes :**
- `n8n_data` : Configuration et credentials n8n
- `./workflows` : Fichiers JSON des workflows

---

### Nginx

| Parametre | Valeur |
|-----------|--------|
| Image | `nginx:alpine` |
| Container | `wibot-nginx` |
| Port expose | `8080:80` |

**Routes configurees :**

| Route | Destination | Description |
|-------|-------------|-------------|
| `/webhook/auth/*` | `n8n:5678` | Authentification |
| `/webhook/wibot/*` | `n8n:5678` | API Chat |
| `/*` | Static files | Frontend React |
| `/health` | 200 OK | Health check |

**Fonctionnalites :**
- Gzip compression
- Rate limiting (10 req/s)
- CORS headers
- Timeout 120s pour requetes AI
- Cache 1 an pour assets statiques

---

## Workflows n8n

### 1. WIBOT - Auth Login

**Endpoint :** `POST /webhook/auth/login`

**Flux :**
```
Webhook → Get User (PostgreSQL) → User Exists?
    ├─ No → User Not Found (401)
    └─ Yes → Verify Password → Password Valid?
                ├─ No → Invalid Password (401)
                └─ Yes → Generate JWT → Success (200)
```

**Request :**
```json
{
  "username": "khora",
  "password": "test123"
}
```

**Response :**
```json
{
  "success": true,
  "token": "eyJhbGc...",
  "user": {
    "id": 1,
    "username": "khora",
    "role": "admin"
  }
}
```

---

### 2. WIBOT - Chat Main

**Endpoint :** `POST /webhook/wibot/chat`

**Flux :**
```
Webhook → Verify JWT → JWT Valid?
    ├─ No → Unauthorized (401)
    └─ Yes → AI Agent (Mistral + Postgres Memory)
                → Format Response → Success (200)
```

**Nodes speciaux :**
- **Mistral Cloud Chat Model** : Modele `mistral-small-latest`, temperature 0.7
- **Postgres Chat Memory** : Session `userId_conversationId`, contexte 10 messages

**Request :**
```json
{
  "conversation_id": "uuid-here",
  "message": "Bonjour, comment ca va ?"
}
```

**Response :**
```json
{
  "success": true,
  "response": "Bonjour ! Je vais bien, merci...",
  "tokens_used": 150,
  "tokens_remaining": 49850,
  "conversation_id": "uuid-here"
}
```

---

### 3. WIBOT - Get Conversations

**Endpoint :** `GET /webhook/wibot/conversations`

**Flux :**
```
Webhook → Verify JWT → JWT Valid?
    ├─ No → Unauthorized (401)
    └─ Yes → Return Empty List → Success (200)
```

**Note :** Retourne une liste vide actuellement. Les conversations sont gerees cote client.

---

### 4. WIBOT - Get Messages

**Endpoint :** `GET /webhook/wibot/conversations/:conversationId/messages`

**Flux :**
```
Webhook → Verify JWT → JWT Valid?
    ├─ No → Unauthorized (401)
    └─ Yes → Get Messages (PostgreSQL n8n_chat_histories)
                → Format Response → Success (200)
```

**Query SQL :**
```sql
SELECT
  id as message_id,
  CASE WHEN message->>'type' = 'human' THEN 'user' ELSE 'assistant' END as role,
  message->>'content' as content
FROM n8n_chat_histories
WHERE session_id = '{userId}_{conversationId}'
ORDER BY id ASC;
```

---

### 5. WIBOT - Create Conversation

**Endpoint :** `POST /webhook/wibot/conversations`

**Flux :**
```
Webhook → Verify JWT & Generate ID → JWT Valid?
    ├─ No → Unauthorized (401)
    └─ Yes → Format Response → Success (201)
```

**Note :** Genere un ID unique sans persister en base.

---

## Schema Base de Donnees

### Table `users`
```sql
CREATE TABLE users (
    user_id         SERIAL PRIMARY KEY,
    username        VARCHAR(100) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    role            VARCHAR(50) DEFAULT 'user',
    created_at      TIMESTAMP DEFAULT NOW(),
    is_active       BOOLEAN DEFAULT true
);
```

### Table `n8n_chat_histories` (creee par n8n)
```sql
CREATE TABLE n8n_chat_histories (
    id          SERIAL PRIMARY KEY,
    session_id  VARCHAR(255),
    message     JSONB
);
```

Format du champ `message` :
```json
{
  "type": "human|ai",
  "content": "Le texte du message",
  "additional_kwargs": {},
  "response_metadata": {}
}
```

---

## Credentials n8n

### PostgreSQL (WIBOT PostgreSQL)
```
Host: postgres
Port: 5432
Database: wibot
User: widip
Password: widip_secure_password_2024
```

### Mistral API
```
API Key: LH0hs7MxTDGx6z2ioHC8oyrFnGHcbywL
```

---

## Securite

### JWT Token
- Algorithme : HS256 (simplifie)
- Duree : 24 heures
- Payload : `userId`, `username`, `role`, `iat`, `exp`

### Rate Limiting (Nginx)
- 10 requetes/seconde par IP
- Burst : 5-10 selon endpoint

### Headers Securite
```
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

---

## Commandes Docker

```bash
# Demarrer tous les services
docker-compose up -d

# Arreter tous les services
docker-compose down

# Voir les logs
docker-compose logs -f
docker-compose logs -f n8n
docker-compose logs -f postgres

# Redemarrer un service
docker-compose restart n8n

# Voir les conteneurs
docker ps

# Executer une commande PostgreSQL
docker exec wibot-postgres psql -U widip -d wibot -c "SELECT * FROM users;"

# Backup base de donnees
docker exec wibot-postgres pg_dump -U widip wibot > backup.sql

# Supprimer les volumes (reset complet)
docker-compose down -v
```

---

## Troubleshooting

### Port deja utilise
```bash
# Changer les ports dans docker-compose.yml
ports:
  - "5433:5432"  # PostgreSQL
  - "5679:5678"  # n8n
  - "8080:80"    # Nginx
```

### n8n ne demarre pas
```bash
# Verifier que PostgreSQL est pret
docker-compose logs postgres
# Attendre "database system is ready to accept connections"
```

### Erreur "relation does not exist"
La table `n8n_chat_histories` est creee automatiquement par le node Postgres Chat Memory lors du premier message.

### Credentials invalides dans n8n
Apres import des workflows, reconfigurer manuellement les credentials PostgreSQL et Mistral dans chaque workflow.
