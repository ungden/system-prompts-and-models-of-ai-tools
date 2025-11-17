# Lovable Clone - Architecture (React + Vite + Supabase)

> **Tech Stack:** Đúng 100% như Lovable thật - React, Vite, Tailwind, Supabase

---

## 📋 Overview

### **Lovable's ACTUAL Tech Stack**

```
Frontend:         React 18 + Vite + TypeScript
Styling:          Tailwind CSS + shadcn/ui
State:            Zustand / Jotai
Backend:          Supabase (Database, Auth, Storage, Realtime, Edge Functions)
AI/LLM:           OpenAI GPT-4 / Anthropic Claude
Real-time:        Supabase Realtime subscriptions
Preview:          WebContainer (StackBlitz) - Live iframe preview
```

### **⚠️ KHÔNG HỖ TRỢ:**
- ❌ Next.js, Angular, Vue, Svelte
- ❌ Backend runtime (Python, Node.js server)
- ❌ Custom API routes

### **✅ HỖ TRỢ:**
- ✅ React + Vite ONLY
- ✅ Supabase Edge Functions (thay API routes)
- ✅ Supabase native integration

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LOVABLE CLONE                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ Sidebar  │  │  Chat Panel  │  │  Live Preview     │   │
│  │          │  │              │  │                   │   │
│  │ Sections │  │  Messages    │  │  ┌─────────────┐ │   │
│  │ Theme    │  │  Input       │  │  │  iframe     │ │   │
│  │ Files    │  │  Streaming   │  │  │  (Vite dev) │ │   │
│  └──────────┘  └──────────────┘  │  └─────────────┘ │   │
│                                   └───────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          ▼
        ┌─────────────────────────────────────┐
        │      Supabase Edge Functions        │
        │  ┌───────────────────────────────┐  │
        │  │ /chat      - AI responses     │  │
        │  │ /codegen   - Generate code    │  │
        │  │ /project   - Project mgmt     │  │
        │  │ /deploy    - Deploy logic     │  │
        │  └───────────────────────────────┘  │
        └─────────────────────────────────────┘
                          ▼
        ┌─────────────────────────────────────┐
        │         Supabase Services           │
        │  • PostgreSQL Database              │
        │  • Auth (email, OAuth)              │
        │  • Storage (file uploads)           │
        │  • Realtime (live updates)          │
        └─────────────────────────────────────┘
```

---

## 🎯 I. FRONTEND LAYER (React + Vite)

### 1. **Project Structure**

```
lovable-clone/
├── src/
│   ├── components/
│   │   ├── chat/
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── ChatMessage.tsx
│   │   │   └── ChatInput.tsx
│   │   ├── preview/
│   │   │   ├── LivePreview.tsx
│   │   │   ├── ConsolePanel.tsx
│   │   │   └── NetworkPanel.tsx
│   │   ├── sidebar/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── SectionsPanel.tsx
│   │   │   ├── ThemePanel.tsx
│   │   │   └── FilesPanel.tsx
│   │   └── ui/              # shadcn/ui components
│   ├── lib/
│   │   ├── supabase.ts       # Supabase client
│   │   ├── ai-agent.ts       # AI agent logic
│   │   └── webcontainer.ts   # WebContainer API
│   ├── stores/
│   │   ├── chat-store.ts     # Zustand store
│   │   ├── preview-store.ts
│   │   └── project-store.ts
│   ├── types/
│   │   └── index.ts
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── supabase/
│   ├── functions/            # Edge Functions
│   │   ├── chat/
│   │   ├── codegen/
│   │   ├── project/
│   │   └── deploy/
│   └── migrations/           # Database schemas
├── public/
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

### 2. **Main App Entry**

**File: `src/main.tsx`**
```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**File: `src/App.tsx`**
```typescript
import { ChatPanel } from './components/chat/ChatPanel';
import { LivePreview } from './components/preview/LivePreview';
import { Sidebar } from './components/sidebar/Sidebar';

export default function App() {
  return (
    <div className="flex h-screen bg-background">
      <Sidebar />

      <div className="flex-1 flex">
        <div className="w-1/2 border-r">
          <ChatPanel />
        </div>

        <div className="w-1/2">
          <LivePreview />
        </div>
      </div>
    </div>
  );
}
```

### 3. **Vite Configuration**

**File: `vite.config.ts`**
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  },
  server: {
    port: 3000,
    headers: {
      'Cross-Origin-Embedder-Policy': 'require-corp',
      'Cross-Origin-Opener-Policy': 'same-origin'
    }
  }
});
```

---

## 🔧 II. SUPABASE BACKEND LAYER

### 1. **Database Schema**

**File: `supabase/migrations/20240101_initial.sql`**
```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users (handled by Supabase Auth)

-- Projects table
CREATE TABLE public.projects (
  id uuid default uuid_generate_v4() primary key,
  user_id uuid references auth.users(id) on delete cascade not null,
  name text not null,
  description text,
  file_tree jsonb default '{}'::jsonb,
  design_system jsonb default '{}'::jsonb,
  dependencies jsonb default '[]'::jsonb,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Messages table (conversation history)
CREATE TABLE public.messages (
  id uuid default uuid_generate_v4() primary key,
  project_id uuid references public.projects(id) on delete cascade not null,
  role text not null check (role in ('user', 'assistant', 'system')),
  content text not null,
  tool_calls jsonb,
  created_at timestamptz default now()
);

-- Files table (generated code)
CREATE TABLE public.project_files (
  id uuid default uuid_generate_v4() primary key,
  project_id uuid references public.projects(id) on delete cascade not null,
  path text not null,
  content text not null,
  language text,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  unique(project_id, path)
);

-- Usage tracking
CREATE TABLE public.usage (
  id uuid default uuid_generate_v4() primary key,
  user_id uuid references auth.users(id) on delete cascade not null,
  tokens integer not null,
  type text not null check (type in ('chat', 'generation')),
  timestamp timestamptz default now()
);

-- Row Level Security
ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.project_files ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.usage ENABLE ROW LEVEL SECURITY;

-- Policies
CREATE POLICY "Users can view own projects"
  ON public.projects FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own projects"
  ON public.projects FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own projects"
  ON public.projects FOR UPDATE
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own projects"
  ON public.projects FOR DELETE
  USING (auth.uid() = user_id);

-- Similar policies for other tables...
```

### 2. **Supabase Client Setup**

**File: `src/lib/supabase.ts`**
```typescript
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseAnonKey);

// Type definitions
export interface Project {
  id: string;
  user_id: string;
  name: string;
  description?: string;
  file_tree: Record<string, any>;
  design_system: Record<string, any>;
  dependencies: string[];
  created_at: string;
  updated_at: string;
}

export interface Message {
  id: string;
  project_id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  tool_calls?: any[];
  created_at: string;
}

export interface ProjectFile {
  id: string;
  project_id: string;
  path: string;
  content: string;
  language?: string;
  created_at: string;
  updated_at: string;
}
```

---

## ⚡ III. SUPABASE EDGE FUNCTIONS (Thay API Routes)

### 1. **Chat Edge Function**

**File: `supabase/functions/chat/index.ts`**
```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import OpenAI from 'https://esm.sh/openai@4';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  // Handle CORS
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    // Get user from auth header
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      {
        global: {
          headers: { Authorization: req.headers.get('Authorization')! },
        },
      }
    );

    const {
      data: { user },
    } = await supabaseClient.auth.getUser();

    if (!user) {
      throw new Error('Not authenticated');
    }

    // Parse request body
    const { message, projectId } = await req.json();

    // Get project and conversation history
    const { data: project } = await supabaseClient
      .from('projects')
      .select('*')
      .eq('id', projectId)
      .single();

    const { data: messages } = await supabaseClient
      .from('messages')
      .select('*')
      .eq('project_id', projectId)
      .order('created_at', { ascending: true });

    // Initialize OpenAI
    const openai = new OpenAI({
      apiKey: Deno.env.get('OPENAI_API_KEY'),
    });

    // Load Lovable system prompt
    const SYSTEM_PROMPT = await Deno.readTextFile(
      './prompts/lovable-system.txt'
    );

    // Build messages array
    const chatMessages = [
      {
        role: 'system',
        content: `${SYSTEM_PROMPT}\n\nProject Context:\n${JSON.stringify(project, null, 2)}`,
      },
      ...messages.map((msg) => ({
        role: msg.role,
        content: msg.content,
      })),
      {
        role: 'user',
        content: message,
      },
    ];

    // Call OpenAI
    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: chatMessages as any,
      temperature: 0.7,
      max_tokens: 4000,
    });

    const response = completion.choices[0].message.content;

    // Save user message
    await supabaseClient.from('messages').insert({
      project_id: projectId,
      role: 'user',
      content: message,
    });

    // Save assistant message
    await supabaseClient.from('messages').insert({
      project_id: projectId,
      role: 'assistant',
      content: response,
    });

    // Track usage
    await supabaseClient.from('usage').insert({
      user_id: user.id,
      tokens: completion.usage?.total_tokens || 0,
      type: 'chat',
    });

    return new Response(JSON.stringify({ response }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 200,
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    });
  }
});
```

### 2. **Code Generation Edge Function**

**File: `supabase/functions/codegen/index.ts`**
```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import OpenAI from 'https://esm.sh/openai@4';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      {
        global: {
          headers: { Authorization: req.headers.get('Authorization')! },
        },
      }
    );

    const {
      data: { user },
    } = await supabaseClient.auth.getUser();

    if (!user) throw new Error('Not authenticated');

    const { requirement, projectId, componentSpec } = await req.json();

    // Get project context
    const { data: project } = await supabaseClient
      .from('projects')
      .select('*')
      .eq('id', projectId)
      .single();

    // Initialize OpenAI
    const openai = new OpenAI({
      apiKey: Deno.env.get('OPENAI_API_KEY'),
    });

    // Load system prompt
    const LOVABLE_PROMPT = await Deno.readTextFile(
      './prompts/lovable-system.txt'
    );

    // Generate code
    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: [
        {
          role: 'system',
          content: LOVABLE_PROMPT,
        },
        {
          role: 'user',
          content: `Generate a React component for: ${requirement}\n\nComponent Spec: ${JSON.stringify(componentSpec)}\n\nProject Context: ${JSON.stringify(project.design_system)}`,
        },
      ],
      temperature: 0.2,
      max_tokens: 4000,
    });

    const generatedCode = completion.choices[0].message.content;

    // Parse code (extract from markdown)
    const codeMatch = generatedCode?.match(/```(?:typescript|tsx)?\n([\s\S]*?)```/);
    const code = codeMatch ? codeMatch[1] : generatedCode;

    // Save file to database
    const filePath = componentSpec.filePath || 'src/components/Generated.tsx';
    await supabaseClient.from('project_files').upsert({
      project_id: projectId,
      path: filePath,
      content: code,
      language: 'typescript',
    });

    return new Response(
      JSON.stringify({
        code,
        filePath,
        success: true,
      }),
      {
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
        status: 200,
      }
    );
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    });
  }
});
```

---

## 🎨 IV. AI AGENT SYSTEM

### 1. **Agent Tools (Lovable Compatible)**

**File: `src/lib/agent-tools.ts`**
```typescript
export interface AgentTool {
  name: string;
  description: string;
  parameters: Record<string, any>;
  execute: (params: any, context: any) => Promise<any>;
}

// Tool: Write File
export const writeFileTool: AgentTool = {
  name: 'lov-write',
  description: 'Write or create a file in the project',
  parameters: {
    file_path: 'string',
    content: 'string',
  },
  execute: async (params, context) => {
    const { file_path, content } = params;

    // Call Supabase Edge Function to save file
    const response = await fetch(
      `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/write-file`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${context.accessToken}`,
        },
        body: JSON.stringify({
          projectId: context.projectId,
          filePath: file_path,
          content,
        }),
      }
    );

    return await response.json();
  },
};

// Tool: Search Files
export const searchFilesTool: AgentTool = {
  name: 'lov-search-files',
  description: 'Search for code patterns in project files',
  parameters: {
    query: 'string (regex pattern)',
    include_pattern: 'string (glob pattern)',
  },
  execute: async (params, context) => {
    const response = await fetch(
      `${import.meta.env.VITE_SUPABASE_URL}/functions/v1/search-files`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${context.accessToken}`,
        },
        body: JSON.stringify({
          projectId: context.projectId,
          ...params,
        }),
      }
    );

    return await response.json();
  },
};

// Tool: Read File
export const readFileTool: AgentTool = {
  name: 'lov-view',
  description: 'Read contents of a file',
  parameters: {
    file_path: 'string',
    lines: 'string (optional)',
  },
  execute: async (params, context) => {
    const { file_path, lines } = params;

    const { data, error } = await context.supabase
      .from('project_files')
      .select('content')
      .eq('project_id', context.projectId)
      .eq('path', file_path)
      .single();

    if (error) throw error;

    let content = data.content;

    if (lines) {
      const [start, end] = lines.split('-').map(Number);
      const allLines = content.split('\n');
      content = allLines.slice(start - 1, end).join('\n');
    }

    return { content, filePath: file_path };
  },
};

export const allTools: AgentTool[] = [
  writeFileTool,
  searchFilesTool,
  readFileTool,
];
```

---

## 🌐 V. WEBCONTAINER INTEGRATION

**File: `src/lib/webcontainer.ts`**
```typescript
import { WebContainer } from '@webcontainer/api';

let webcontainerInstance: WebContainer;

export async function bootWebContainer(): Promise<WebContainer> {
  if (webcontainerInstance) return webcontainerInstance;

  webcontainerInstance = await WebContainer.boot();
  return webcontainerInstance;
}

export async function createViteProject(
  projectName: string,
  files: Record<string, string>
): Promise<string> {
  const container = await bootWebContainer();

  // Create project structure for Vite
  const fileTree: any = {
    'package.json': {
      file: {
        contents: JSON.stringify(
          {
            name: projectName,
            private: true,
            version: '0.0.0',
            type: 'module',
            scripts: {
              dev: 'vite',
              build: 'vite build',
              preview: 'vite preview',
            },
            dependencies: {
              react: '^18.3.0',
              'react-dom': '^18.3.0',
            },
            devDependencies: {
              '@types/react': '^18.3.0',
              '@types/react-dom': '^18.3.0',
              '@vitejs/plugin-react': '^4.3.0',
              typescript: '^5.5.0',
              vite: '^5.4.0',
              tailwindcss: '^3.4.0',
              autoprefixer: '^10.4.0',
              postcss: '^8.4.0',
            },
          },
          null,
          2
        ),
      },
    },
    'index.html': {
      file: {
        contents: `<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>${projectName}</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>`,
      },
    },
    'vite.config.ts': {
      file: {
        contents: `import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})`,
      },
    },
    src: {
      directory: {},
    },
  };

  // Add user files
  for (const [path, content] of Object.entries(files)) {
    const parts = path.split('/');
    let current = fileTree;

    for (let i = 0; i < parts.length - 1; i++) {
      if (!current[parts[i]]) {
        current[parts[i]] = { directory: {} };
      }
      current = current[parts[i]].directory;
    }

    current[parts[parts.length - 1]] = {
      file: { contents: content },
    };
  }

  // Mount files
  await container.mount(fileTree);

  // Install dependencies
  const installProcess = await container.spawn('npm', ['install']);
  await installProcess.exit;

  // Start dev server
  const devProcess = await container.spawn('npm', ['run', 'dev']);

  // Wait for server
  container.on('server-ready', (port, url) => {
    console.log('Vite dev server ready at:', url);
  });

  return container.url;
}
```

---

## 📊 VI. STATE MANAGEMENT (Zustand)

**File: `src/stores/chat-store.ts`**
```typescript
import { create } from 'zustand';
import { Message } from '@/types';

interface ChatStore {
  messages: Message[];
  projectId: string;
  isStreaming: boolean;

  addMessage: (message: Message) => void;
  updateMessage: (id: string, updates: Partial<Message>) => void;
  clearMessages: () => void;
  setProjectId: (id: string) => void;
  setStreaming: (isStreaming: boolean) => void;
}

export const useChatStore = create<ChatStore>((set) => ({
  messages: [],
  projectId: '',
  isStreaming: false,

  addMessage: (message) =>
    set((state) => ({
      messages: [...state.messages, message],
    })),

  updateMessage: (id, updates) =>
    set((state) => ({
      messages: state.messages.map((msg) =>
        msg.id === id ? { ...msg, ...updates } : msg
      ),
    })),

  clearMessages: () => set({ messages: [] }),
  setProjectId: (id) => set({ projectId: id }),
  setStreaming: (isStreaming) => set({ isStreaming }),
}));
```

**File: `src/stores/preview-store.ts`**
```typescript
import { create } from 'zustand';

interface PreviewStore {
  url: string;
  consoleLogs: any[];
  networkRequests: any[];

  setUrl: (url: string) => void;
  addConsoleLog: (log: any) => void;
  addNetworkRequest: (request: any) => void;
  clearLogs: () => void;
  reload: () => void;
}

export const usePreviewStore = create<PreviewStore>((set, get) => ({
  url: '',
  consoleLogs: [],
  networkRequests: [],

  setUrl: (url) => set({ url }),

  addConsoleLog: (log) =>
    set((state) => ({
      consoleLogs: [...state.consoleLogs, log],
    })),

  addNetworkRequest: (request) =>
    set((state) => ({
      networkRequests: [...state.networkRequests, request],
    })),

  clearLogs: () => set({ consoleLogs: [], networkRequests: [] }),

  reload: () => {
    const currentUrl = get().url;
    set({ url: '' });
    setTimeout(() => set({ url: currentUrl }), 100);
  },
}));
```

---

## 🔐 VII. AUTHENTICATION (Supabase Auth)

**File: `src/lib/auth.ts`**
```typescript
import { supabase } from './supabase';

export async function signUp(email: string, password: string) {
  const { data, error } = await supabase.auth.signUp({
    email,
    password,
  });

  if (error) throw error;
  return data;
}

export async function signIn(email: string, password: string) {
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) throw error;
  return data;
}

export async function signOut() {
  const { error } = await supabase.auth.signOut();
  if (error) throw error;
}

export async function getUser() {
  const {
    data: { user },
  } = await supabase.auth.getUser();
  return user;
}

// GitHub OAuth
export async function signInWithGitHub() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'github',
  });

  if (error) throw error;
  return data;
}
```

---

## 🚀 VIII. DEPLOYMENT

### Recommended Stack:
- **Frontend (Vite)**: Vercel / Netlify / Cloudflare Pages
- **Backend (Supabase)**: Supabase Cloud (auto-handled)
- **Edge Functions**: Deploy via Supabase CLI

### Deploy Commands:
```bash
# Build Vite app
npm run build

# Deploy Edge Functions
supabase functions deploy chat
supabase functions deploy codegen
supabase functions deploy project

# Deploy frontend to Vercel
vercel --prod
```

---

## 📝 SUMMARY

This architecture is **100% compatible** with Lovable's actual stack:

✅ **Frontend**: React + Vite + Tailwind
✅ **Backend**: Supabase ONLY (no custom servers)
✅ **API Layer**: Edge Functions (not Next.js API routes)
✅ **Preview**: WebContainer for live Vite preview
✅ **Real-time**: Supabase Realtime subscriptions
✅ **AI**: OpenAI/Anthropic via Edge Functions

❌ **NOT SUPPORTED**: Next.js, backend runtimes, custom API routes

---

**Next Steps**: See implementation guides for code examples!
