cd "C:\Users\maxim\Desktop\Projet IA\WIDIP\Widip Clé 25-12-2025\Projet WIDIP IA\Constructions\WIBOT\wibot-frontend"
npm run dev

navigateur :
localStorage.setItem('wibot_token', 'test-token');
localStorage.setItem('wibot_user', JSON.stringify({id: 1, username: 'Test User', role: 'admin'}));
location.reload();

arborecence frontend : 
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