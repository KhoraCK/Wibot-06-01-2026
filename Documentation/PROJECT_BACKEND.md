# WIBOT Backend - Documentation Référence

## Vue d'ensemble
Backend pour chatbot d'entreprise WIDIP : n8n + PostgreSQL + AI Agent Mistral avec mémoire contextuelle.

**Stack** : n8n (workflows) + PostgreSQL 14+ + Azure Mistral API + JWT Auth

---

## 🔗 Intégration Frontend

Ce backend communique avec le frontend React WIBOT (voir PROJECT.md dans le dossier frontend).

**Contrat strict** : Tous les endpoints DOIVENT respecter exactement les formats TypeScript définis dans la section "Contrat API Frontend" ci-dessous. Toute déviation causera des erreurs d'intégration.

**Tests d'intégration** : Après implémentation, tester avec curl en simulant exactement les requêtes du frontend.

---

## Architecture Globale
```
Frontend React (3000)
        ↓ HTTPS
Nginx Reverse Proxy
        ↓
n8n Webhooks (5678)
    ├── Auth → PostgreSQL
    ├── Chat → AI Agent (Mistral) → PostgreSQL
    └── Conversations → PostgreSQL
        ↓
Azure Mistral API
```

---

## ⚠️ CONTRAT API AVEC FRONTEND

Le backend DOIT respecter exactement ces formats pour communiquer avec le frontend React.

### Types TypeScript (Référence Frontend)
```typescript
// ============================================
// AUTHENTIFICATION
// ============================================

interface LoginRequest {
  username: string;
  password: string;
}

interface LoginResponse {
  success: boolean;
  token: string;
  user: {
    id: number;
    username: string;
    role: string;
  };
}

interface LoginErrorResponse {
  success: false;
  error: string;  // Message clair pour affichage
}

// ============================================
// CHAT
// ============================================

interface ChatRequest {
  conversation_id: string;  // UUID format
  message: string;
  files?: FileAttachment[];  // Optionnel
}

interface FileAttachment {
  name: string;      // "document.pdf"
  content: string;   // Contenu texte ou base64
}

interface ChatResponse {
  success: boolean;
  response: string;              // Réponse de l'assistant
  tokens_used: number;           // Tokens consommés ce message
  tokens_remaining: number;      // Tokens restants ce mois
  conversation_id: string;       // Même UUID que request
}

interface ChatErrorResponse {
  success: false;
  error: string;
  code?: string;  // "QUOTA_EXCEEDED", "INVALID_TOKEN", etc.
}

// ============================================
// CONVERSATIONS
// ============================================

interface ConversationsResponse {
  conversations: Conversation[];
}

interface Conversation {
  conversation_id: string;  // UUID
  title: string;            // Max 255 chars
  updated_at: string;       // ISO 8601 format "2024-12-25T14:30:00Z"
  message_count: number;
}

// ============================================
// MESSAGES
// ============================================

interface MessagesResponse {
  messages: Message[];
}

interface Message {
  message_id: number;
  role: 'user' | 'assistant';  // Exactement ces valeurs
  content: string;
  created_at: string;          // ISO 8601 format
}

// ============================================
// ERREURS HTTP
// ============================================

// 401 Unauthorized
{
  "success": false,
  "error": "Token invalide ou expiré"
}

// 429 Too Many Requests (quota dépassé)
{
  "success": false,
  "error": "Quota tokens mensuel dépassé. Limite : 50,000 tokens/mois.",
  "code": "QUOTA_EXCEEDED"
}

// 500 Server Error
{
  "success": false,
  "error": "Erreur serveur. Réessayez dans quelques instants."
}
```

### Points Critiques d'Alignement

#### 1. Format Dates
✅ **TOUJOURS** ISO 8601 : `2024-12-25T14:30:00Z`  
❌ **JAMAIS** timestamps UNIX ou formats locaux

**PostgreSQL** : Utiliser `TO_CHAR(created_at, 'YYYY-MM-DD"T"HH24:MI:SS"Z"')` ou fonction équivalente

#### 2. UUID Conversations
✅ Format : `550e8400-e29b-41d4-a716-446655440000`  
✅ Type PostgreSQL : `UUID`  
✅ Génération : `gen_random_uuid()` côté DB ou frontend génère avec `crypto.randomUUID()`

#### 3. Role Messages
✅ **Exactement** : `'user'` ou `'assistant'`  
❌ Pas `'human'`, `'ai'`, `'bot'`, etc.

#### 4. Tokens
✅ `tokens_used` : tokens consommés pour ce message uniquement  
✅ `tokens_remaining` : quota_tokens - used_tokens du mois en cours  
❌ Ne pas confondre avec `total_tokens` de l'API Mistral

#### 5. Success Flag
✅ Toujours inclure `"success": true/false` dans toutes les responses  
✅ Si `success: false` → inclure `"error": "message clair et actionnable"`

#### 6. Headers HTTP
**Request Frontend** :
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Response Backend** :
```
Content-Type: application/json
```

#### 7. Files Upload
Frontend envoie :
```json
{
  "conversation_id": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Analyse ce document",
  "files": [
    {
      "name": "rapport.pdf",
      "content": "Contenu texte extrait du PDF..."
    }
  ]
}
```

Backend doit :
- Accepter array `files` (optionnel, peut être absent ou vide)
- Concaténer `files[].content` au contexte du message avant envoi à AI Agent
- **NE PAS** sauvegarder les fichiers sur disque (juste utiliser pour contexte)
- Optionnel : sauvegarder métadonnées dans `messages.attachments` (JSONB)

---

## ✅ Checklist Validation Contrat

Avant de valider le backend, tester :

- [ ] Login renvoie bien `{success, token, user: {id, username, role}}`
- [ ] JWT payload contient `user_id` (pas `id` ou `userId`)
- [ ] Chat response contient `tokens_remaining` (pas `remaining_tokens`)
- [ ] Toutes les dates au format ISO 8601 avec "Z"
- [ ] Conversation UUID valide (36 chars avec tirets)
- [ ] Role messages = `'user'` | `'assistant'` exactement (vérifier CHECK constraint)
- [ ] Toutes les erreurs incluent `success: false` + `error: "message"`
- [ ] Files array accepté dans ChatRequest (optionnel)
- [ ] 401 si JWT expiré/invalide
- [ ] 429 si quota dépassé avec message clair et code "QUOTA_EXCEEDED"

---

## Schéma PostgreSQL

### Tables

**users**
```sql
user_id         SERIAL PRIMARY KEY
username        VARCHAR(100) UNIQUE NOT NULL
password_hash   VARCHAR(255) NOT NULL  -- bcrypt hash
email           VARCHAR(255)
role            VARCHAR(50) DEFAULT 'user'
created_at      TIMESTAMP DEFAULT NOW()
is_active       BOOLEAN DEFAULT true
```

**conversations**
```sql
conversation_id  UUID PRIMARY KEY DEFAULT gen_random_uuid()
user_id          INTEGER REFERENCES users(user_id)
title            VARCHAR(255)
created_at       TIMESTAMP DEFAULT NOW()
updated_at       TIMESTAMP DEFAULT NOW()
```

**messages**
```sql
message_id       SERIAL PRIMARY KEY
conversation_id  UUID REFERENCES conversations(conversation_id) ON DELETE CASCADE
user_id          INTEGER REFERENCES users(user_id)
role             VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant'))
content          TEXT NOT NULL
tokens           INTEGER DEFAULT 0
attachments      JSONB  -- Métadonnées fichiers optionnel
created_at       TIMESTAMP DEFAULT NOW()
```

**user_token_usage**
```sql
usage_id        SERIAL PRIMARY KEY
user_id         INTEGER REFERENCES users(user_id)
month           DATE NOT NULL
used_tokens     BIGINT DEFAULT 0
quota_tokens    BIGINT DEFAULT 50000
UNIQUE(user_id, month)
```

### Index Critiques
```sql
CREATE INDEX idx_conversations_user ON conversations(user_id, updated_at DESC);
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at DESC);
CREATE INDEX idx_messages_user ON messages(user_id);
CREATE INDEX idx_token_usage_user_month ON user_token_usage(user_id, month);
```

### Trigger auto-update
```sql
-- Fonction pour auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger sur conversations
CREATE TRIGGER update_conversation_timestamp
BEFORE UPDATE ON conversations
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

---

## API Endpoints (n8n Webhooks)

⚠️ **Respecter strictement les formats définis dans "Contrat API Frontend"**

### 1. POST /webhook/auth/login

**Request** : Voir `LoginRequest` (Contrat API)  
**Response Success** : Voir `LoginResponse`  
**Response Error** : Voir `LoginErrorResponse`

**Exemple Request** :
```json
{
  "username": "khora",
  "password": "test123"
}
```

**Exemple Success (200)** :
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "username": "khora",
    "role": "admin"
  }
}
```

**Exemple Error (401)** :
```json
{
  "success": false,
  "error": "Identifiants invalides"
}
```

**Logique** :
1. Valider input (username + password présents)
2. PostgreSQL : SELECT user WHERE username = $1 AND is_active = true
3. bcrypt.compare(password, password_hash)
4. Si OK : générer JWT (HS256, 8h expiration) avec payload `{user_id, username, role}`
5. Retourner token + user data

---

### 2. POST /webhook/wibot/chat

**Headers** : `Authorization: Bearer {JWT}`  
**Request** : Voir `ChatRequest` (Contrat API)  
**Response Success** : Voir `ChatResponse`  
**Response Error** : Voir `ChatErrorResponse`

**Exemple Request** :
```json
{
  "conversation_id": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Bonjour WIBOT !",
  "files": [
    {
      "name": "log.txt",
      "content": "[2024-12-25] Error: Connection timeout..."
    }
  ]
}
```

**Exemple Success (200)** :
```json
{
  "success": true,
  "response": "Bonjour ! Je vois que tu as un problème de timeout. Laisse-moi analyser le log...",
  "tokens_used": 145,
  "tokens_remaining": 49855,
  "conversation_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Exemple Error Quota (429)** :
```json
{
  "success": false,
  "error": "Quota tokens mensuel dépassé. Limite : 50,000 tokens/mois.",
  "code": "QUOTA_EXCEEDED"
}
```

**Logique** :
1. Verify JWT → extraire user_id
2. Check quota : SELECT user_token_usage WHERE user_id + month
   - Si used_tokens >= quota_tokens → 429
3. Load historique : SELECT 10 derniers messages (role, content) ORDER BY created_at ASC
4. Préparer contexte :
   - Si files présents : concaténer `files[].content` au message
   - Formater historique pour AI Agent
5. **AI Agent call** (Mistral) avec system prompt + historique + message
6. Extraire réponse + tokens_used (de l'API ou estimer)
7. INSERT messages :
   - Message user (role='user', content=message original)
   - Message assistant (role='assistant', content=réponse AI)
8. UPSERT conversation (créer si n'existe pas, sinon UPDATE updated_at)
   - title = SUBSTRING(message, 1, 255) si nouvelle conv
9. UPDATE user_token_usage (UPSERT) : used_tokens += tokens_used
10. Calculer tokens_remaining = quota_tokens - used_tokens
11. Retourner response

---

### 3. GET /webhook/wibot/conversations

**Headers** : `Authorization: Bearer {JWT}`  
**Response** : Voir `ConversationsResponse` (Contrat API)

**Exemple Success (200)** :
```json
{
  "conversations": [
    {
      "conversation_id": "550e8400-e29b-41d4-a716-446655440000",
      "title": "Problème timeout réseau",
      "updated_at": "2024-12-25T14:30:00Z",
      "message_count": 8
    },
    {
      "conversation_id": "660f9511-f3ac-52e5-b827-557766551111",
      "title": "Script Python backup",
      "updated_at": "2024-12-24T09:15:00Z",
      "message_count": 3
    }
  ]
}
```

**Logique** :
1. Verify JWT → user_id
2. PostgreSQL :
```sql
   SELECT 
     c.conversation_id,
     c.title,
     TO_CHAR(c.updated_at, 'YYYY-MM-DD"T"HH24:MI:SS"Z"') as updated_at,
     COUNT(m.message_id) as message_count
   FROM conversations c
   LEFT JOIN messages m ON c.conversation_id = m.conversation_id
   WHERE c.user_id = $1
   GROUP BY c.conversation_id, c.title, c.updated_at
   ORDER BY c.updated_at DESC
   LIMIT 50
```
3. Retourner array conversations

---

### 4. GET /webhook/wibot/conversations/:id/messages

**Headers** : `Authorization: Bearer {JWT}`  
**URL Param** : `id` = conversation_id (UUID)  
**Response** : Voir `MessagesResponse` (Contrat API)

**Exemple Success (200)** :
```json
{
  "messages": [
    {
      "message_id": 1,
      "role": "user",
      "content": "Bonjour WIBOT",
      "created_at": "2024-12-25T10:00:00Z"
    },
    {
      "message_id": 2,
      "role": "assistant",
      "content": "Bonjour ! Comment puis-je t'aider ?",
      "created_at": "2024-12-25T10:00:02Z"
    }
  ]
}
```

**Exemple Error (403)** :
```json
{
  "success": false,
  "error": "Vous n'avez pas accès à cette conversation"
}
```

**Logique** :
1. Verify JWT → user_id
2. Vérifier ownership : 
```sql
   SELECT user_id FROM conversations WHERE conversation_id = $1
```
   - Si user_id != JWT user_id → 403
3. SELECT messages :
```sql
   SELECT 
     message_id,
     role,
     content,
     TO_CHAR(created_at, 'YYYY-MM-DD"T"HH24:MI:SS"Z"') as created_at
   FROM messages
   WHERE conversation_id = $1
   ORDER BY created_at ASC
```
4. Retourner array messages

---

## Configuration AI Agent (Node n8n)

### Chat Model
- **Type** : Mistral Cloud API (ou Azure OpenAI avec endpoint Mistral si Azure)
- **Model** : `mistral-large-latest` (ou `devstral-2` si disponible)
- **Credential** : Mistral API Key

### Parameters
- **Temperature** : 0.7
- **Max Tokens** : 2000
- **Top P** : 1
- **Frequency Penalty** : 0
- **Presence Penalty** : 0

### Memory
- **Type** : Window Buffer Memory
- **Window Size** : 10 messages
- **Session ID** : `{{ $json.conversation_id }}`
- **Memory Key** : "chat_history"

### System Prompt
```
Tu es WIBOT, l'assistant IA de WIDIP, une coopérative française (SCOP) qui fournit des services d'infrastructure IT à plus de 600 établissements de santé (EHPAD, cliniques, associations).

## Ton rôle
Tu assistes les 20 collaborateurs WIDIP (techniciens support, admins sys, ingénieurs réseau) dans leurs tâches quotidiennes :
- Support technique (GLPI, infrastructure réseau, Observium)
- Rédaction de documentation technique
- Aide au développement (Python, JavaScript, SQL, Bash, n8n)
- Résumés de tickets, incidents ou conversations
- Conseils IT et bonnes pratiques
- Automatisation de tâches répétitives

## Contexte WIDIP
- Secteur : Santé (certifications HDS + ISO 27001 critiques)
- Stack technique : GLPI (ticketing), Observium (monitoring réseau), n8n (workflows), PostgreSQL
- Volume : ~20,000 tickets support/an
- Infrastructure : on-premise + quelques services cloud

## Règles strictes
✅ Sois précis, professionnel et concis
✅ Adapte ton niveau technique à l'interlocuteur
✅ Si tu ne connais pas une info spécifique WIDIP, dis-le clairement
✅ Respecte la confidentialité des données santé (RGPD/HDS)
✅ Propose des solutions pragmatiques adaptées au contexte WIDIP
✅ Cite tes sources si tu références une documentation externe

❌ N'invente JAMAIS d'information sur l'infrastructure WIDIP
❌ Ne génère pas de credentials, mots de passe ou données sensibles
❌ Ne suggère pas de solutions non conformes RGPD/HDS
❌ N'accède pas à des données patient (même en exemple)

## Ton ton
Amical mais professionnel. Tu tutoies les collaborateurs (culture SCOP).
Tu peux utiliser des emojis occasionnellement pour rendre l'échange plus chaleureux.

Date actuelle : {{ new Date().toISOString().split('T')[0] }}
```

---

## Structure Workflows n8n

### Workflow 1 : auth_login
```
Webhook POST /webhook/auth/login
    ↓
Function: Validate Input (username + password présents)
    ↓
PostgreSQL: Get User (SELECT WHERE username)
    ↓
IF: User Found?
    ↓ YES
Function: Verify Password (bcrypt.compare)
    ↓
IF: Password Valid?
    ↓ YES
Function: Generate JWT (jsonwebtoken, 8h expire)
    ↓
Respond: {success: true, token, user}
    
    ↓ NO (any IF)
Respond: {success: false, error: "..."}
```

### Workflow 2 : chat_main (⭐ avec AI Agent)
```
Webhook POST /webhook/wibot/chat
    ↓
Function: Verify JWT (extract user_id)
    ↓
PostgreSQL: Check Token Quota
    ↓
IF: Quota OK?
    ↓ YES
PostgreSQL: Load Last 10 Messages (role, content)
    ↓
Function: Prepare Context (merge files if present)
    ↓
AI Agent (Mistral)
  - System prompt WIBOT
  - Memory: Window Buffer (10 msgs)
  - Input: formatted message
    ↓
Function: Extract Response & Calculate Tokens
    ↓
PostgreSQL: INSERT User Message (role='user')
    ↓
PostgreSQL: INSERT Assistant Message (role='assistant')
    ↓
PostgreSQL: UPSERT Conversation (title, updated_at)
    ↓
PostgreSQL: UPDATE Token Usage (UPSERT)
    ↓
Function: Format Response (tokens_remaining = quota - used)
    ↓
Respond: {success: true, response, tokens_used, tokens_remaining, conversation_id}

    ↓ NO (quota)
Respond 429: {success: false, error: "Quota dépassé", code: "QUOTA_EXCEEDED"}
```

### Workflow 3 : get_conversations
```
Webhook GET /webhook/wibot/conversations
    ↓
Function: Verify JWT
    ↓
PostgreSQL: SELECT Conversations + Message Count
  - WHERE user_id
  - ORDER BY updated_at DESC
  - Format dates ISO 8601
    ↓
Respond: {conversations: [...]}
```

### Workflow 4 : get_messages
```
Webhook GET /webhook/wibot/conversations/:id/messages
    ↓
Function: Verify JWT
    ↓
PostgreSQL: Verify Ownership (conversation belongs to user)
    ↓
IF: Authorized?
    ↓ YES
PostgreSQL: SELECT Messages
  - WHERE conversation_id
  - ORDER BY created_at ASC
  - Format dates ISO 8601
    ↓
Respond: {messages: [...]}

    ↓ NO
Respond 403: {success: false, error: "Accès refusé"}
```

---

## Sécurité

### JWT
- **Secret** : Min 32 chars aléatoires (env: JWT_SECRET)
- **Algorithm** : HS256
- **Expiration** : 8h
- **Payload** : `{user_id: number, username: string, role: string, iat: number, exp: number}`

### Passwords
- **Hashing** : bcrypt (rounds: 10)
- **Storage** : `password_hash` dans table users
- **Never** log ou retourner passwords en clair

### SQL
- **TOUJOURS** parameterized queries (`$1`, `$2`, etc.)
- **JAMAIS** string interpolation directe

### Errors
- **200** : Success
- **400** : Bad Request (validation failed)
- **401** : Unauthorized (JWT invalide/expiré)
- **403** : Forbidden (pas propriétaire de la ressource)
- **429** : Too Many Requests (quota dépassé)
- **500** : Server Error

---

## Variables Environnement
```bash
# JWT
JWT_SECRET=your-super-secret-key-minimum-32-characters-random

# Mistral API
MISTRAL_API_KEY=your-azure-mistral-api-key

# PostgreSQL
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=wibot
POSTGRES_USER=widip
POSTGRES_PASSWORD=your-secure-password
```

---

## Credentials n8n à créer

### 1. PostgreSQL
- **Type** : PostgreSQL
- **Name** : "WIBOT PostgreSQL"
- **Host** : `${POSTGRES_HOST}` (ou IP)
- **Port** : `${POSTGRES_PORT}`
- **Database** : `${POSTGRES_DB}`
- **User** : `${POSTGRES_USER}`
- **Password** : `${POSTGRES_PASSWORD}`
- **SSL** : Disabled (réseau interne)

### 2. Mistral API
- **Type** : Mistral Cloud API (ou "HTTP Header Auth" si pas de node dédié)
- **Name** : "Mistral API"
- **API Key** : `${MISTRAL_API_KEY}`

---

## Données de Test

### User test
```sql
-- Username: khora
-- Password: test123
-- Générer hash avec: node -e "console.log(require('bcrypt').hashSync('test123', 10))"

INSERT INTO users (username, password_hash, email, role)
VALUES (
  'khora',
  '$2b$10$...',  -- À remplacer par le hash réel
  'khora@widip.fr',
  'admin'
);
```

### Conversation + Messages test
```sql
-- Conversation test
INSERT INTO conversations (conversation_id, user_id, title, created_at, updated_at)
VALUES (
  '550e8400-e29b-41d4-a716-446655440000',
  1,
  'Conversation de test',
  NOW() - INTERVAL '2 days',
  NOW() - INTERVAL '1 day'
);

-- Messages test (5-10 messages alternés)
INSERT INTO messages (conversation_id, user_id, role, content, tokens, created_at) VALUES
('550e8400-e29b-41d4-a716-446655440000', 1, 'user', 'Bonjour WIBOT !', 10, NOW() - INTERVAL '2 days'),
('550e8400-e29b-41d4-a716-446655440000', 1, 'assistant', 'Bonjour ! Comment puis-je t''aider ?', 15, NOW() - INTERVAL '2 days' + INTERVAL '2 seconds'),
('550e8400-e29b-41d4-a716-446655440000', 1, 'user', 'Peux-tu m''expliquer comment fonctionne PostgreSQL ?', 12, NOW() - INTERVAL '2 days' + INTERVAL '1 minute'),
('550e8400-e29b-41d4-a716-446655440000', 1, 'assistant', 'PostgreSQL est un système de gestion de base de données relationnelle...', 85, NOW() - INTERVAL '2 days' + INTERVAL '1 minute 5 seconds'),
('550e8400-e29b-41d4-a716-446655440000', 1, 'user', 'Et comment faire un backup ?', 8, NOW() - INTERVAL '1 day'),
('550e8400-e29b-41d4-a716-446655440000', 1, 'assistant', 'Pour faire un backup PostgreSQL, tu peux utiliser pg_dump...', 95, NOW() - INTERVAL '1 day' + INTERVAL '3 seconds');
```

### Token usage test
```sql
INSERT INTO user_token_usage (user_id, month, used_tokens, quota_tokens)
VALUES (
  1,
  DATE_TRUNC('month', CURRENT_DATE),
  12450,
  50000
);
```

---

## Requêtes SQL Utiles

### Login
```sql
SELECT user_id, username, password_hash, email, role
FROM users
WHERE username = $1 AND is_active = true
LIMIT 1;
```

### Check Quota
```sql
SELECT used_tokens, quota_tokens
FROM user_token_usage
WHERE user_id = $1 
  AND month = DATE_TRUNC('month', CURRENT_DATE)
LIMIT 1;
```

### Load Chat History (mémoire)
```sql
SELECT role, content
FROM messages
WHERE conversation_id = $1
ORDER BY created_at ASC
LIMIT 10;
```

### List Conversations avec compteur
```sql
SELECT 
  c.conversation_id,
  c.title,
  TO_CHAR(c.updated_at, 'YYYY-MM-DD"T"HH24:MI:SS"Z"') as updated_at,
  COUNT(m.message_id) as message_count
FROM conversations c
LEFT JOIN messages m ON c.conversation_id = m.conversation_id
WHERE c.user_id = $1
GROUP BY c.conversation_id, c.title, c.updated_at
ORDER BY c.updated_at DESC
LIMIT 50;
```

### Get Messages avec dates formatées
```sql
SELECT 
  message_id,
  role,
  content,
  TO_CHAR(created_at, 'YYYY-MM-DD"T"HH24:MI:SS"Z"') as created_at
FROM messages
WHERE conversation_id = $1
ORDER BY created_at ASC;
```

### Update Tokens (UPSERT)
```sql
INSERT INTO user_token_usage (user_id, month, used_tokens, quota_tokens)
VALUES ($1, DATE_TRUNC('month', CURRENT_DATE), $2, 50000)
ON CONFLICT (user_id, month)
DO UPDATE SET used_tokens = user_token_usage.used_tokens + $2
RETURNING used_tokens, quota_tokens;
```

### Verify Conversation Ownership
```sql
SELECT user_id
FROM conversations
WHERE conversation_id = $1;
```

### UPSERT Conversation
```sql
INSERT INTO conversations (conversation_id, user_id, title, created_at, updated_at)
VALUES ($1, $2, $3, NOW(), NOW())
ON CONFLICT (conversation_id)
DO UPDATE SET updated_at = NOW();
```

---

## Tests Validation

### Setup: Récupérer JWT
```bash
TOKEN=$(curl -s -X POST http://localhost:5678/webhook/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "khora", "password": "test123"}' \
  | jq -r '.token')

echo "Token: $TOKEN"
```

### Test 1: Login Success
```bash
curl -X POST http://localhost:5678/webhook/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "khora", "password": "test123"}'

# Expected:
# {
#   "success": true,
#   "token": "eyJhbG...",
#   "user": {"id": 1, "username": "khora", "role": "admin"}
# }
```

### Test 2: Login Fail
```bash
curl -X POST http://localhost:5678/webhook/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "khora", "password": "wrongpass"}'

# Expected:
# {
#   "success": false,
#   "error": "Identifiants invalides"
# }
```

### Test 3: Chat Simple
```bash
curl -X POST http://localhost:5678/webhook/wibot/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "550e8400-e29b-41d4-a716-446655440000",
    "message": "Salut WIBOT ! Comment ça va ?"
  }'

# Expected:
# {
#   "success": true,
#   "response": "Salut ! Ça va très bien...",
#   "tokens_used": 45,
#   "tokens_remaining": 49955,
#   "conversation_id": "550e8400-e29b-41d4-a716-446655440000"
# }
```

### Test 4: Chat avec Mémoire
```bash
# Premier message
curl -X POST http://localhost:5678/webhook/wibot/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "660f9511-f3ac-52e5-b827-557766551111",
    "message": "Parle-moi de PostgreSQL"
  }'

# Deuxième message (doit se souvenir du contexte)
curl -X POST http://localhost:5678/webhook/wibot/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "660f9511-f3ac-52e5-b827-557766551111",
    "message": "Comment je fais un backup ?"
  }'

# WIBOT doit comprendre qu'on parle de backup PostgreSQL ✓
```

### Test 5: Chat avec Fichier
```bash
curl -X POST http://localhost:5678/webhook/wibot/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "770f9622-g4bd-63f6-c938-668877662222",
    "message": "Analyse ce log et dis-moi ce qui ne va pas",
    "files": [
      {
        "name": "error.log",
        "content": "[2024-12-25 14:30:15] ERROR: Connection timeout to database\n[2024-12-25 14:30:16] ERROR: Retry failed"
      }
    ]
  }'

# WIBOT doit analyser le log et diagnostiquer le problème ✓
```

### Test 6: Get Conversations
```bash
curl -X GET http://localhost:5678/webhook/wibot/conversations \
  -H "Authorization: Bearer $TOKEN"

# Expected:
# {
#   "conversations": [
#     {
#       "conversation_id": "550e8400-...",
#       "title": "Conversation de test",
#       "updated_at": "2024-12-24T10:00:00Z",
#       "message_count": 6
#     }
#   ]
# }
```

### Test 7: Get Messages
```bash
curl -X GET "http://localhost:5678/webhook/wibot/conversations/550e8400-e29b-41d4-a716-446655440000/messages" \
  -H "Authorization: Bearer $TOKEN"

# Expected:
# {
#   "messages": [
#     {
#       "message_id": 1,
#       "role": "user",
#       "content": "Bonjour WIBOT !",
#       "created_at": "2024-12-23T10:00:00Z"
#     },
#     ...
#   ]
# }
```

### Test 8: Quota Exceeded (simuler)
```bash
# Modifier manuellement used_tokens en DB pour dépasser quota
# UPDATE user_token_usage SET used_tokens = 51000 WHERE user_id = 1;

curl -X POST http://localhost:5678/webhook/wibot/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "550e8400-e29b-41d4-a716-446655440000",
    "message": "Test quota"
  }'

# Expected 429:
# {
#   "success": false,
#   "error": "Quota tokens mensuel dépassé. Limite : 50,000 tokens/mois.",
#   "code": "QUOTA_EXCEEDED"
# }
```

---

## Notes Importantes

⚠️ **AI Agent** : Gère automatiquement la mémoire contextuelle via Window Buffer Memory  
⚠️ **Mémoire** : 10 derniers messages chargés de PostgreSQL + passés à l'agent  
⚠️ **Tools** : Vides en Phase 1, MCP servers ajoutés en Phase 2  
⚠️ **Tokens** : Si AI Agent n'expose pas usage, estimer : `Math.ceil((input + output).length / 4)`  
⚠️ **Files** : Concaténer le contenu au message, pas de stockage fichier  
⚠️ **Dates** : TOUJOURS ISO 8601 avec TO_CHAR PostgreSQL  
⚠️ **Role** : CHECK constraint sur 'user' | 'assistant' strictement  

---

## Phase 2 (Futur) - MCP Integration

Quand MCP servers seront prêts :

**Dans AI Agent → Tools**, ajouter :
- **GLPI Tool** : Search tickets, create ticket, update status, get ticket details
- **Observium Tool** : Get device status, network alerts, bandwidth metrics
- **Active Directory Tool** : Search users, groups, get user info
- **Email Tool (SMTP)** : Send formatted emails, notifications

L'AI Agent pourra automatiquement appeler ces tools selon le contexte :
```
User: "Crée un ticket GLPI pour le switch core en panne"
→ Agent: appelle GLPI Tool.create_ticket()
→ Répond: "Ticket #12345 créé avec priorité urgente"

User: "Quel est le statut du serveur backup ?"
→ Agent: appelle Observium Tool.get_device_status("backup-srv")
→ Répond: "Le serveur backup est UP, CPU 15%, RAM 45%"
```

---

## Checkpoints Validation Backend

Après Phase 1 (PostgreSQL) :
- [ ] Database `wibot` créée
- [ ] 4 tables créées avec bon schéma
- [ ] Index optimisation présents
- [ ] Trigger updated_at fonctionne
- [ ] Données test insérées (user + conv + messages)
- [ ] Requêtes test passent

Après Phase 2 (Auth) :
- [ ] Workflow auth_login importé dans n8n
- [ ] Credential PostgreSQL configurée
- [ ] JWT généré avec bon format
- [ ] Login success retourne {success, token, user}
- [ ] Login fail retourne {success: false, error}
- [ ] bcrypt compare fonctionne

Après Phase 3 (Chat) :
- [ ] Workflow chat_main importé
- [ ] Credential Mistral API configurée
- [ ] AI Agent node configuré (model, prompt, memory)
- [ ] Chat retourne réponse Mistral
- [ ] Mémoire contextuelle fonctionne (10 msgs)
- [ ] Messages sauvegardés en DB
- [ ] Compteur tokens s'incrémente
- [ ] Quota check bloque si dépassé (429)
- [ ] Files uploadés traités correctement
- [ ] Workflows conversations + messages fonctionnent
- [ ] Formats JSON matchent exactement le contrat frontend

---

**Version** : 2.0 - Backend WIBOT avec Contrat API Frontend  
**Date** : Décembre 2024  
**Responsable** : Khora - WIDIP  
**Intégration** : Frontend React (voir PROJECT.md)