# WIBOT Frontend - Documentation Technique

## Vue d'ensemble

Le frontend WIBOT est une application React moderne offrant une interface de chat similaire a Claude.ai. Il permet aux employes WIDIP d'interagir avec un assistant IA propulse par Mistral.

## Stack Technique

| Technologie | Version | Role |
|-------------|---------|------|
| React | 19.2 | Framework UI |
| TypeScript | 5.9 | Typage statique |
| Vite | 7.2 | Build tool / Dev server |
| Tailwind CSS | 3.4 | Framework CSS |
| Zustand | 5.0 | State management |
| React Router | 7.11 | Routing SPA |
| Axios | 1.13 | Client HTTP |
| Lucide React | 0.562 | Icones |
| React Markdown | 10.1 | Rendu Markdown |
| React Syntax Highlighter | 16.1 | Coloration syntaxique |

---

## Architecture des dossiers

```
wibot-frontend/
├── public/                    # Assets statiques
├── src/
│   ├── components/           # Composants React
│   │   ├── chat/            # Composants chat
│   │   │   ├── ChatWindow.tsx      # Zone d'affichage des messages
│   │   │   ├── Message.tsx         # Composant message individuel
│   │   │   ├── MarkdownRenderer.tsx # Rendu Markdown
│   │   │   ├── CodeBlock.tsx       # Blocs de code avec syntax highlighting
│   │   │   └── index.ts
│   │   ├── layout/          # Layout principal
│   │   │   ├── Header.tsx          # En-tete avec user menu et tokens
│   │   │   ├── Sidebar.tsx         # Barre laterale conversations
│   │   │   ├── InputBar.tsx        # Zone de saisie message
│   │   │   └── index.ts
│   │   ├── ui/              # Composants UI reutilisables
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Spinner.tsx
│   │   │   ├── Toast.tsx
│   │   │   ├── Skeleton.tsx
│   │   │   └── index.ts
│   │   ├── upload/          # Upload de fichiers
│   │   │   ├── FileDropzone.tsx
│   │   │   ├── FilePreview.tsx
│   │   │   └── index.ts
│   │   └── ProtectedRoute.tsx
│   ├── hooks/               # Custom hooks
│   │   ├── useAuth.ts            # Authentification
│   │   ├── useChat.ts            # Envoi de messages
│   │   ├── useConversations.ts   # Gestion conversations
│   │   ├── useToast.ts           # Notifications
│   │   └── index.ts
│   ├── pages/               # Pages de l'application
│   │   ├── Login.tsx             # Page de connexion
│   │   └── Chat.tsx              # Page principale chat
│   ├── services/            # Services API
│   │   ├── api.ts                # Client Axios configure
│   │   └── auth.ts               # Gestion tokens localStorage
│   ├── store/               # State management Zustand
│   │   ├── index.ts              # Auth, Conversations, Chat stores
│   │   └── toastStore.ts         # Store notifications
│   ├── styles/
│   │   └── index.css             # Styles globaux + Tailwind
│   ├── types/
│   │   └── index.ts              # Types TypeScript
│   ├── utils/
│   │   └── date.ts               # Formatage dates
│   ├── App.tsx                   # Composant racine + routing
│   └── main.tsx                  # Point d'entree
├── index.html
├── package.json
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

---

## Design System

### Palette de couleurs

```css
--bg-primary: #1A1A1A;        /* Fond principal */
--bg-secondary: #141414;       /* Fond secondaire (sidebar, header) */
--bg-user-msg: #2A2A2A;        /* Fond messages utilisateur */
--bg-assistant-msg: #232323;   /* Fond messages assistant */
--bg-code: #0D0D0D;            /* Fond blocs de code */
--text-primary: #E5E5E5;       /* Texte principal */
--text-secondary: #A0A0A0;     /* Texte secondaire */
--accent: #5B9EFF;             /* Couleur accent (bleu) */
--accent-hover: #4A8DE8;       /* Accent hover */
--border: #333333;             /* Bordures */
```

### Dimensions fixes

```css
--sidebar-width: 280px;
--header-height: 60px;
--input-bar-height: 80px;
```

---

## Flux de donnees

### Authentification

```
Login.tsx
  └─> useAuth.ts (hook)
        └─> api.ts POST /webhook/auth/login
              └─> auth.ts (saveToken, saveUser)
                    └─> store/index.ts (setAuth)
                          └─> Redirect /chat
```

### Envoi de message

```
InputBar.tsx (handleSubmit)
  └─> useChat.ts (sendMessage)
        └─> api.ts POST /webhook/wibot/chat
              └─> store (addMessage user)
              └─> Reponse AI
                    └─> store (addMessage assistant)
```

### Gestion des conversations

```
Sidebar.tsx
  ├─> createConversation() → Nouveau UUID client
  └─> selectConversation() → GET /webhook/wibot/conversations/:id/messages
```

---

## Stores Zustand

### AuthStore
```typescript
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  setAuth: (user: User, token: string) => void;
  clearAuth: () => void;
  initAuth: () => void;  // Restore from localStorage
}
```

### ConversationsStore
```typescript
interface ConversationsState {
  conversations: Conversation[];
  currentConversation: Conversation | null;
  setConversations: (conversations: Conversation[]) => void;
  setCurrentConversation: (conversation: Conversation | null) => void;
  addConversation: (conversation: Conversation) => void;
  updateConversation: (id: string, updates: Partial<Conversation>) => void;
}
```

### ChatStore
```typescript
interface ChatState {
  messages: Message[];
  isLoading: boolean;
  tokensRemaining: number;
  setMessages: (messages: Message[]) => void;
  addMessage: (message: Message) => void;
  setLoading: (loading: boolean) => void;
  setTokensRemaining: (tokens: number) => void;
  clearMessages: () => void;
}
```

---

## Endpoints API consommes

| Methode | Endpoint | Description |
|---------|----------|-------------|
| POST | `/webhook/auth/login` | Authentification |
| GET | `/webhook/wibot/conversations` | Liste conversations |
| POST | `/webhook/wibot/conversations` | Creer conversation |
| GET | `/webhook/wibot/conversations/:id/messages` | Messages d'une conversation |
| POST | `/webhook/wibot/chat` | Envoyer message au chatbot |

---

## Fonctionnalites

### Interface Chat
- Affichage messages avec avatars distincts (user/assistant)
- Rendu Markdown complet (titres, listes, tableaux, liens)
- Blocs de code avec coloration syntaxique
- Bouton copier le code
- Auto-scroll vers le dernier message
- Indicateur "WIBOT reflechit..."

### Gestion Conversations
- Liste des conversations dans la sidebar
- Creation nouvelle conversation
- Selection et chargement des messages
- Titre auto-genere depuis le premier message

### Authentification
- Page login avec validation
- JWT stocke en localStorage
- Interceptor Axios pour ajouter le token
- Redirect automatique si token expire (401)

### Upload Fichiers
- Drag & drop zone
- Preview des fichiers selectionnes
- Formats acceptes : PDF, TXT, MD, CSV, JSON
- Limite : 10 Mo par fichier

### Compteur Tokens
- Affichage tokens restants dans le header
- Barre de progression coloree
- Alertes si tokens bas ou epuises

---

## Commandes

```bash
# Installation des dependances
npm install

# Demarrer en mode developpement
npm run dev

# Build de production
npm run build

# Preview du build
npm run preview

# Linting
npm run lint
```

---

## Configuration

### Variables d'environnement (.env)

```env
VITE_API_URL=http://localhost:8080
VITE_MAX_FILE_SIZE=10485760
```

### vite.config.ts

Le build est optimise avec des chunks separes :
- `react-vendor` : React, ReactDOM, React Router
- `ui-vendor` : Lucide, Zustand
- `markdown` : React Markdown, Remark GFM
- `syntax-highlighter` : React Syntax Highlighter

---

## Notes importantes

1. **Le frontend est une SPA** - Nginx doit rediriger toutes les routes vers index.html
2. **Les conversation_id sont generes cote client** avec `crypto.randomUUID()`
3. **Le token JWT expire apres 24h** - l'utilisateur sera redirige vers /login
4. **Le build de production** doit etre copie dans `wibot-backend/frontend/` pour etre servi par Nginx
