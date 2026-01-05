# WIBOT - Documentation Projet Frontend

## Vision
Application web React pour chatbot d'entreprise WIDIP. Interface professionnelle type Claude.ai, hébergée localement, connectée au backend n8n existant.

## Stack Technique Validée
- **Framework** : React 18 + TypeScript
- **Build** : Vite
- **Styling** : Tailwind CSS
- **State** : Zustand (simple et performant)
- **Routing** : React Router v6
- **HTTP** : Axios
- **Markdown** : react-markdown + remark-gfm
- **Code Highlighting** : react-syntax-highlighter
- **Icons** : Lucide React
- **Upload** : react-dropzone

## Architecture Backend (Existant - NE PAS TOUCHER)
```
Backend n8n (PostgreSQL)
├── POST /webhook/auth/login → {token, user}
├── POST /webhook/wibot/chat → {response, tokens_used, tokens_remaining}
├── GET /webhook/wibot/conversations → {conversations[]}
└── GET /webhook/wibot/conversations/:id/messages → {messages[]}
```

## Types Backend (Référence)
```typescript
// Auth
interface LoginResponse {
  success: boolean;
  token: string;
  user: { id: number; username: string; role: string };
}

// Chat
interface ChatRequest {
  conversation_id: string;
  message: string;
  files?: { name: string; content: string }[];
}

interface ChatResponse {
  success: boolean;
  response: string;
  tokens_used: number;
  tokens_remaining: number;
  conversation_id: string;
}

// Conversations
interface Conversation {
  conversation_id: string;
  title: string;
  updated_at: string;
  message_count: number;
}

// Messages
interface Message {
  message_id: number;
  role: 'user' | 'assistant';
  content: string;
  created_at: string;
}
```

## Structure Projet
```
wibot-frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── pages/
│   │   ├── Login.tsx
│   │   └── Chat.tsx
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── InputBar.tsx
│   │   ├── chat/
│   │   │   ├── ChatWindow.tsx
│   │   │   ├── Message.tsx
│   │   │   ├── CodeBlock.tsx
│   │   │   └── MarkdownRenderer.tsx
│   │   ├── upload/
│   │   │   ├── FileDropzone.tsx
│   │   │   └── FilePreview.tsx
│   │   └── ui/
│   │       ├── Button.tsx
│   │       ├── Input.tsx
│   │       └── Spinner.tsx
│   ├── services/
│   │   ├── api.ts
│   │   └── auth.ts
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useConversations.ts
│   │   └── useChat.ts
│   ├── store/
│   │   └── index.ts
│   ├── types/
│   │   └── index.ts
│   └── styles/
│       └── index.css
├── package.json
├── vite.config.ts
├── tailwind.config.js
└── tsconfig.json
```

## Design System

### Couleurs (Mode Sombre)
```css
--bg-primary: #1A1A1A
--bg-secondary: #141414
--bg-user-msg: #2A2A2A
--bg-assistant-msg: #232323
--bg-code: #0D0D0D
--text-primary: #E5E5E5
--text-secondary: #A0A0A0
--accent: #5B9EFF
--accent-hover: #4A8DE8
--border: #333333
```

### Typographie
```css
font-family: Inter, 'Segoe UI', system-ui, sans-serif
font-size-base: 15px
font-size-code: 14px
```

### Spacing
```
sidebar-width: 280px
header-height: 60px
input-bar-height: 80px
```

## Layout Desktop
```
┌─────────────────────────────────────────────────┐
│ Header (60px) - Logo + Tokens + User            │
├─────────────┬───────────────────────────────────┤
│             │                                   │
│  Sidebar    │    Chat Area                      │
│  (280px)    │    (flex-1)                       │
│             │                                   │
│ [+ New]     │    [Messages scroll]              │
│ Conv 1      │                                   │
│ Conv 2      │                                   │
│             │    [Input Bar - 80px]             │
└─────────────┴───────────────────────────────────┘
```

## Règles de Développement

### Code Quality
- ✅ TypeScript strict mode
- ✅ Composants fonctionnels uniquement
- ✅ Hooks pour logique réutilisable
- ✅ Props typées avec interfaces
- ✅ Gestion erreurs avec try/catch
- ✅ Loading states partout
- ✅ Pas de console.log en prod
- ❌ Pas de any
- ❌ Pas de class components
- ❌ Pas de logique dans JSX

### Performance
- Bundle < 2MB
- Lazy loading pages
- Memo composants lourds
- Debounce inputs si nécessaire

### Sécurité
- JWT dans localStorage (interne OK)
- Sanitize user input
- Validate file types/sizes
- HTTPS uniquement en prod

## Variables Environnement
```
VITE_API_URL=https://n8n.widip.local
VITE_MAX_FILE_SIZE=10485760
VITE_APP_NAME=WIBOT
```

## Dépendances Autorisées
```json
{
  "react": "^18.2.0",
  "react-dom": "^18.2.0",
  "react-router-dom": "^6.20.0",
  "zustand": "^4.4.0",
  "axios": "^1.6.0",
  "react-markdown": "^9.0.0",
  "remark-gfm": "^4.0.0",
  "react-syntax-highlighter": "^15.5.0",
  "react-dropzone": "^14.2.0",
  "lucide-react": "^0.294.0",
  "date-fns": "^2.30.0"
}
```

## Checkpoints de Validation

Après chaque phase, vérifier :
- [ ] Code compile sans erreurs TypeScript
- [ ] npm run dev lance l'app
- [ ] Fonctionnalités phase testées manuellement
- [ ] Pas de console errors
- [ ] Design conforme mockups

## Notes Importantes

⚠️ **Backend est DÉJÀ fait** - Ne pas générer de code backend
⚠️ **Local uniquement** - Pas de services cloud
⚠️ **Pas de WebSocket** Phase 1 - HTTP classique suffit
⚠️ **Mémoire contextuelle** gérée par backend - Frontend envoie juste conversation_id

## Prochaines Étapes (Post-MVP)
- WebSocket pour streaming réponses
- PWA pour installation desktop-like
- Raccourcis clavier avancés
- Thème clair
- Export conversations
```

---

# 🚀 PROMPTS SÉQUENTIELS (4 phases)

## PHASE 1 : Setup & Architecture
```
# PHASE 1 - WIBOT Setup & Architecture

CONTEXTE : Je développe WIBOT (voir PROJECT.md pour détails complets).

OBJECTIF PHASE 1 : Créer la structure complète du projet avec configuration et composants de base UI.

TÂCHES :

1. **Initialiser projet Vite + React + TypeScript**
```bash
   npm create vite@latest wibot-frontend -- --template react-ts
```

2. **Installer dépendances**
```bash
   npm install react-router-dom zustand axios react-markdown remark-gfm react-syntax-highlighter react-dropzone lucide-react date-fns
   npm install -D tailwindcss postcss autoprefixer @types/react-syntax-highlighter
   npx tailwindcss init -p
```

3. **Configurer Tailwind** (tailwind.config.js)
   - Ajouter palette couleurs custom du PROJECT.md
   - Configure content paths

4. **Créer structure dossiers** (voir PROJECT.md)
   - Tous les dossiers vides avec .gitkeep

5. **Créer types globaux** (src/types/index.ts)
   - Copier types du PROJECT.md
   - Ajouter types UI (Button, Input props)

6. **Créer composants UI de base** (src/components/ui/)
   - Button.tsx : props {variant, size, onClick, children, disabled, loading}
   - Input.tsx : props {type, value, onChange, placeholder, error}
   - Spinner.tsx : animation loading simple

7. **Setup styles globaux** (src/styles/index.css)
   - Import Tailwind
   - Variables CSS custom du PROJECT.md
   - Reset/normalize basique

8. **Créer .env.example**
```
   VITE_API_URL=http://localhost:5678
   VITE_MAX_FILE_SIZE=10485760
   VITE_APP_NAME=WIBOT
```

9. **README.md basique**
   - Installation
   - Dev (npm run dev)
   - Build (npm run build)

LIVRABLE :
- Projet qui compile (npm run dev fonctionne)
- Page blanche avec "WIBOT" affiché
- Composants UI de base stylés Tailwind

⏸️ PAUSE APRÈS CETTE PHASE - J'attends validation avant Phase 2.
```

---

## PHASE 2 : Auth + Navigation
```
# PHASE 2 - WIBOT Auth & Navigation

PRÉ-REQUIS : Phase 1 terminée et validée. Lire PROJECT.md pour contexte.

OBJECTIF PHASE 2 : Implémenter authentification complète et routing.

TÂCHES :

1. **Service API** (src/services/api.ts)
   - Axios instance avec baseURL depuis env
   - Interceptor request : ajouter JWT si présent
   - Interceptor response : gérer 401 (logout auto)
   - Export fonctions : login(), getConversations(), etc.

2. **Service Auth** (src/services/auth.ts)
   - saveToken(), getToken(), removeToken() (localStorage)
   - isAuthenticated() : boolean

3. **Hook useAuth** (src/hooks/useAuth.ts)
   - State : user, isLoading, error
   - Fonctions : login(username, password), logout()
   - Persist user dans localStorage après login

4. **Store Zustand** (src/store/index.ts)
   - Slice auth : user, setUser, clearUser
   - Slice conversations : conversations[], setConversations
   - Slice chat : currentConversation, messages[], addMessage

5. **Page Login** (src/pages/Login.tsx)
   - Formulaire centré : username + password
   - Bouton submit avec loading state
   - Error message si login fail
   - Redirect vers /chat si success
   - Design : card centrée, fond gradient subtil

6. **Page Chat** (src/pages/Chat.tsx)
   - Layout : Header + Sidebar + ChatArea
   - Pour l'instant juste structure vide
   - Protected route (redirect /login si pas auth)

7. **Header Component** (src/components/layout/Header.tsx)
   - Logo WIBOT gauche
   - Compteur tokens droite (hardcodé "0 / 50,000" pour l'instant)
   - User menu dropdown : username + bouton Logout

8. **Setup Router** (src/App.tsx)
   - Route / → redirect /chat
   - Route /login → Login page
   - Route /chat → Chat page (protected)
   - ProtectedRoute wrapper component

LIVRABLE :
- Login fonctionnel (appel API backend)
- JWT sauvegardé après login
- Redirection /chat si authentifié
- Header affiché avec user + logout
- Chat page (vide mais structure OK)

TESTS MANUELS :
- Login avec user test → redirige /chat ✓
- Logout → redirige /login ✓
- Accès /chat sans auth → redirige /login ✓
- JWT persist après refresh page ✓

⏸️ PAUSE APRÈS CETTE PHASE - J'attends validation avant Phase 3.
```

---

## PHASE 3 : Chat Core
```
# PHASE 3 - WIBOT Chat Core

PRÉ-REQUIS : Phase 2 validée. Lire PROJECT.md.

OBJECTIF PHASE 3 : Implémenter le cœur fonctionnel du chat.

TÂCHES :

1. **Hook useConversations** (src/hooks/useConversations.ts)
   - loadConversations() : GET /webhook/wibot/conversations
   - createConversation() : génère UUID, ajoute au store
   - selectConversation(id) : charge messages de cette conv

2. **Hook useChat** (src/hooks/useChat.ts)
   - sendMessage(message, files?) : POST /webhook/wibot/chat
   - State : isLoading, error
   - Update store avec nouveau message user + assistant

3. **Sidebar Component** (src/components/layout/Sidebar.tsx)
   - Bouton "+ Nouvelle conversation" en haut
   - Liste conversations scrollable
   - Item conversation : title (ellipsis) + date relative
   - Highlight conversation active
   - Click → selectConversation()

4. **ChatWindow Component** (src/components/chat/ChatWindow.tsx)
   - Scroll container auto-scroll vers bas
   - Map messages → Message component
   - Loading indicator si isLoading
   - Empty state si aucun message

5. **Message Component** (src/components/chat/Message.tsx)
   - Props : {role, content, timestamp}
   - Layout différent si user vs assistant
   - Avatar (U ou W selon role)
   - Timestamp formaté (date-fns)
   - Si assistant : utiliser MarkdownRenderer

6. **MarkdownRenderer Component** (src/components/chat/MarkdownRenderer.tsx)
   - react-markdown avec remark-gfm
   - Custom components pour code blocks
   - Détection ```language → CodeBlock component

7. **CodeBlock Component** (src/components/chat/CodeBlock.tsx)
   - react-syntax-highlighter (theme vscode dark)
   - Bouton copier (lucide-react Copy icon)
   - Feedback copie : icône Check 2s
   - Language badge optionnel

8. **InputBar Component** (src/components/layout/InputBar.tsx)
   - Textarea auto-resize (max 5 lignes)
   - Bouton Send (disabled si vide ou loading)
   - Placeholder : "Votre message..."
   - Enter → send (Shift+Enter → newline)
   - Clear après envoi

9. **Assembler Chat.tsx**
   - useEffect : loadConversations() au mount
   - Layout final : Header + Sidebar + ChatWindow + InputBar
   - Passer props aux enfants

LIVRABLE :
- Conversations chargées depuis backend
- Créer nouvelle conversation fonctionnel
- Envoyer message → appel backend → affiche réponse
- Markdown rendu correctement
- Code blocks avec syntax highlighting + copier
- Auto-scroll messages
- Interface complète et fonctionnelle

TESTS MANUELS :
- Créer conv → UUID généré ✓
- Envoyer message → réponse affichée ✓
- Markdown (bold, liste, liens) rendu ✓
- Code block Python copié ✓
- Scroll auto vers bas après message ✓

⏸️ PAUSE APRÈS CETTE PHASE - J'attends validation avant Phase 4.
```

---

## PHASE 4 : Features Avancées
```
# PHASE 4 - WIBOT Features Avancées

PRÉ-REQUIS : Phase 3 validée. Chat core fonctionnel.

OBJECTIF PHASE 4 : Ajouter upload fichiers, compteur tokens live, polish UI.

TÂCHES :

1. **FileDropzone Component** (src/components/upload/FileDropzone.tsx)
   - react-dropzone
   - Zone drag & drop visuelle
   - Validation : formats (.pdf, .txt, .md, .csv, .json)
   - Validation : taille max 10MB
   - Emit onFilesSelected(files)

2. **FilePreview Component** (src/components/upload/FilePreview.tsx)
   - Props : {files, onRemove}
   - Liste fichiers avec icône + nom + taille
   - Bouton X pour retirer

3. **Intégrer upload dans InputBar**
   - Bouton paperclip ouvre FileDropzone
   - Files preview au-dessus textarea
   - Lire fichiers avec FileReader
   - Envoyer dans ChatRequest.files

4. **Compteur tokens live**
   - Récupérer tokens_remaining de ChatResponse
   - Update Header après chaque message
   - Afficher "12,450 / 50,000"
   - Warning si < 5000 (couleur orange)
   - Bloquer send si 0 (message clair)

5. **Error handling robuste**
   - Toast notifications (créer Toast component)
   - Erreurs API affichées (401, 429, 500, timeout)
   - Retry automatique si timeout
   - Messages clairs pour user

6. **Polish UI**
   - Animations transitions (fade-in messages)
   - Hover states tous boutons/links
   - Focus states accessibilité
   - Scrollbar custom (Tailwind scrollbar plugin)
   - Loading skeletons conversations

7. **Format dates relatif**
   - Util formatRelativeDate() avec date-fns
   - "Il y a 2min", "Hier 14h32", "15 déc 2024"
   - Séparateurs date dans ChatWindow

8. **Optimisations**
   - React.memo sur Message component
   - useMemo pour listes triées
   - Lazy load pages (React.lazy)

9. **Documentation finale**
   - README complet
   - DEPLOYMENT.md (Docker + nginx)
   - Commenter code complexe

LIVRABLE :
- Upload fichiers drag & drop fonctionnel
- Fichiers envoyés au backend
- Compteur tokens live et précis
- Toasts pour erreurs
- UI polie et fluide
- App production-ready

TESTS FINAUX :
- Upload PDF → contenu extracté ✓
- Quota tokens affiché correctement ✓
- Erreur 401 → logout auto ✓
- Erreur 429 → message quota dépassé ✓
- Animations fluides ✓
- Build prod (npm run build) sans warnings ✓

✅ PROJET COMPLET - Prêt pour déploiement Docker WIDIP.