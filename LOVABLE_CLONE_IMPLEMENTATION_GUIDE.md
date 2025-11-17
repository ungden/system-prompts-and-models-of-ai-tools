# 🚀 Lovable Clone - Implementation Guide (React + Vite)

> Step-by-step để build Lovable Clone ĐÚNG tech stack - React + Vite + Supabase Edge Functions

---

## 📋 What We're Building

### Core Features (MVP)
1. ✅ **Chat Interface** - Chat với AI để generate code
2. ✅ **AI Agent** - Process requests và generate React components
3. ✅ **File Manager** - Display và manage project files
4. ✅ **Live Preview** - Preview app trong iframe (WebContainer)
5. ✅ **Code Editor** - Edit code trực tiếp
6. ✅ **Project Management** - Save/load projects

### Tech Stack (Đúng như Lovable!)
```
Frontend:  React 18 + Vite + TypeScript + Tailwind CSS
Backend:   Supabase Edge Functions (Deno runtime)
AI:        OpenAI GPT-4 hoặc Anthropic Claude
Preview:   WebContainer (StackBlitz) - Live Vite preview
Database:  Supabase PostgreSQL
Auth:      Supabase Auth
State:     Zustand
```

---

## 🎯 DAY 1: Setup Project

### Step 1: Create Vite Project

```bash
# Create Vite + React + TypeScript project
npm create vite@latest lovable-clone -- --template react-ts
cd lovable-clone
```

### Step 2: Install Dependencies

```bash
# Core dependencies
npm install @supabase/supabase-js
npm install zustand
npm install react-markdown
npm install lucide-react

# WebContainer for live preview
npm install @webcontainer/api

# UI Components (shadcn/ui)
npm install @radix-ui/react-dialog
npm install @radix-ui/react-dropdown-menu
npm install @radix-ui/react-scroll-area
npm install @radix-ui/react-tabs
npm install class-variance-authority clsx tailwind-merge

# Code editor
npm install @monaco-editor/react

# Tailwind
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 3: Configure Tailwind

**File: `tailwind.config.js`**
```js
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

**File: `src/index.css`**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Step 4: Setup Environment Variables

**File: `.env`**
```env
# Supabase
VITE_SUPABASE_URL=https://xxxxx.supabase.co
VITE_SUPABASE_ANON_KEY=eyJxxx...

# OpenAI
VITE_OPENAI_API_KEY=sk-...
```

### Step 5: Project Structure

```bash
mkdir -p src/{components/{chat,preview,sidebar,editor,ui},lib,stores,types}
mkdir -p supabase/{functions,migrations}
```

Final structure:
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
│   │   │   └── ConsolePanel.tsx
│   │   ├── sidebar/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── SectionsPanel.tsx
│   │   │   └── FilesPanel.tsx
│   │   ├── editor/
│   │   │   └── CodeEditor.tsx
│   │   └── ui/
│   ├── lib/
│   │   ├── supabase.ts
│   │   ├── webcontainer.ts
│   │   └── agent-tools.ts
│   ├── stores/
│   │   ├── chat-store.ts
│   │   ├── preview-store.ts
│   │   └── project-store.ts
│   ├── types/
│   │   └── index.ts
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── supabase/
│   ├── functions/
│   │   ├── chat/
│   │   └── codegen/
│   └── migrations/
├── index.html
├── vite.config.ts
├── tailwind.config.js
└── package.json
```

---

## 🎯 DAY 2: Database Setup

### Step 1: Create Supabase Project

1. Go to https://supabase.com
2. Create new project
3. Copy URL and anon key to `.env`

### Step 2: Database Schema

**File: `supabase/migrations/001_initial.sql`**

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Projects table
CREATE TABLE public.projects (
  id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  name text NOT NULL,
  description text,
  file_tree jsonb DEFAULT '{}'::jsonb,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

-- Project files
CREATE TABLE public.project_files (
  id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
  project_id uuid REFERENCES public.projects(id) ON DELETE CASCADE NOT NULL,
  path text NOT NULL,
  content text NOT NULL,
  language text,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  UNIQUE(project_id, path)
);

-- Chat messages
CREATE TABLE public.messages (
  id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
  project_id uuid REFERENCES public.projects(id) ON DELETE CASCADE NOT NULL,
  role text NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
  content text NOT NULL,
  created_at timestamptz DEFAULT now()
);

-- Enable Row Level Security
ALTER TABLE public.projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.project_files ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.messages ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users can manage own projects"
  ON public.projects FOR ALL
  USING (auth.uid() = user_id);

CREATE POLICY "Users can manage project files"
  ON public.project_files FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM public.projects p
      WHERE p.id = project_files.project_id
      AND p.user_id = auth.uid()
    )
  );

CREATE POLICY "Users can manage messages"
  ON public.messages FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM public.projects p
      WHERE p.id = messages.project_id
      AND p.user_id = auth.uid()
    )
  );
```

Run migration in Supabase Dashboard SQL Editor.

### Step 3: Supabase Client

**File: `src/lib/supabase.ts`**

```typescript
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseAnonKey);

// Types
export interface Project {
  id: string;
  user_id: string;
  name: string;
  description?: string;
  file_tree: Record<string, any>;
  created_at: string;
  updated_at: string;
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

export interface Message {
  id: string;
  project_id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  created_at: string;
}
```

---

## 🎯 DAY 3: Main Layout

**File: `src/App.tsx`**

```typescript
import { ChatPanel } from './components/chat/ChatPanel';
import { LivePreview } from './components/preview/LivePreview';
import { Sidebar } from './components/sidebar/Sidebar';

export default function App() {
  return (
    <div className="flex h-screen bg-gray-50">
      {/* Sidebar */}
      <Sidebar />

      {/* Main Content */}
      <div className="flex-1 flex">
        {/* Chat Panel */}
        <div className="w-1/2 border-r bg-white">
          <ChatPanel />
        </div>

        {/* Live Preview */}
        <div className="w-1/2 bg-white">
          <LivePreview />
        </div>
      </div>
    </div>
  );
}
```

---

## 🎯 DAY 4: Chat Interface

**File: `src/components/chat/ChatPanel.tsx`**

```typescript
import { useState, useRef, useEffect } from 'react';
import { useChatStore } from '@/stores/chat-store';
import { ChatMessage } from './ChatMessage';
import { Send } from 'lucide-react';

export function ChatPanel() {
  const [input, setInput] = useState('');
  const scrollRef = useRef<HTMLDivElement>(null);
  const { messages, addMessage, sendMessage, isLoading } = useChatStore();

  useEffect(() => {
    scrollRef.current?.scrollTo(0, scrollRef.current.scrollHeight);
  }, [messages]);

  const handleSend = async () => {
    if (!input.trim() || isLoading) return;

    const userMessage = {
      id: Date.now().toString(),
      role: 'user' as const,
      content: input,
      created_at: new Date().toISOString(),
    };

    addMessage(userMessage);
    setInput('');
    await sendMessage(input);
  };

  return (
    <div className="flex flex-col h-full">
      {/* Header */}
      <div className="p-4 border-b">
        <h2 className="text-lg font-semibold">Chat</h2>
        <p className="text-sm text-gray-500">Describe what you want to build</p>
      </div>

      {/* Messages */}
      <div ref={scrollRef} className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg) => (
          <ChatMessage key={msg.id} message={msg} />
        ))}
      </div>

      {/* Input */}
      <div className="p-4 border-t">
        <div className="flex gap-2">
          <textarea
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                handleSend();
              }
            }}
            placeholder="Describe your app..."
            className="flex-1 p-3 border rounded-lg resize-none focus:outline-none focus:ring-2 focus:ring-blue-500"
            rows={3}
            disabled={isLoading}
          />
          <button
            onClick={handleSend}
            disabled={!input.trim() || isLoading}
            className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <Send className="w-5 h-5" />
          </button>
        </div>
      </div>
    </div>
  );
}
```

**File: `src/components/chat/ChatMessage.tsx`**

```typescript
import { Message } from '@/types';
import ReactMarkdown from 'react-markdown';

interface Props {
  message: Message;
}

export function ChatMessage({ message }: Props) {
  const isUser = message.role === 'user';

  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'}`}>
      <div
        className={`max-w-[80%] rounded-lg p-3 ${
          isUser
            ? 'bg-blue-500 text-white'
            : 'bg-gray-100 text-gray-900'
        }`}
      >
        {isUser ? (
          <p className="whitespace-pre-wrap">{message.content}</p>
        ) : (
          <div className="prose prose-sm max-w-none">
            <ReactMarkdown>{message.content}</ReactMarkdown>
          </div>
        )}
      </div>
    </div>
  );
}
```

**File: `src/stores/chat-store.ts`**

```typescript
import { create } from 'zustand';
import { supabase } from '@/lib/supabase';
import { Message } from '@/types';

interface ChatStore {
  messages: Message[];
  projectId: string;
  isLoading: boolean;

  addMessage: (message: Message) => void;
  sendMessage: (content: string) => Promise<void>;
  setProjectId: (id: string) => void;
}

export const useChatStore = create<ChatStore>((set, get) => ({
  messages: [],
  projectId: '',
  isLoading: false,

  addMessage: (message) =>
    set((state) => ({ messages: [...state.messages, message] })),

  sendMessage: async (content) => {
    set({ isLoading: true });

    try {
      // Call Supabase Edge Function
      const { data, error } = await supabase.functions.invoke('chat', {
        body: {
          message: content,
          projectId: get().projectId,
        },
      });

      if (error) throw error;

      const assistantMessage: Message = {
        id: (Date.now() + 1).toString(),
        role: 'assistant',
        content: data.response,
        created_at: new Date().toISOString(),
      };

      get().addMessage(assistantMessage);
    } catch (error) {
      console.error('Error sending message:', error);
    } finally {
      set({ isLoading: false });
    }
  },

  setProjectId: (id) => set({ projectId: id }),
}));
```

---

## 🎯 DAY 5: Supabase Edge Function (AI Chat)

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

    const { data: { user } } = await supabaseClient.auth.getUser();
    if (!user) throw new Error('Not authenticated');

    const { message, projectId } = await req.json();

    // Get conversation history
    const { data: messages } = await supabaseClient
      .from('messages')
      .select('*')
      .eq('project_id', projectId)
      .order('created_at', { ascending: true })
      .limit(20);

    // Initialize OpenAI
    const openai = new OpenAI({
      apiKey: Deno.env.get('OPENAI_API_KEY'),
    });

    const SYSTEM_PROMPT = `You are Lovable, an AI assistant that helps users build web applications using React, Vite, and Tailwind CSS.

You can:
- Generate React components
- Fix bugs and errors
- Provide coding guidance
- Create responsive UI layouts

Always respond in a helpful and concise manner. When generating code, use modern React patterns with TypeScript and Tailwind CSS.`;

    // Build messages
    const chatMessages = [
      { role: 'system', content: SYSTEM_PROMPT },
      ...(messages || []).map((msg) => ({
        role: msg.role,
        content: msg.content,
      })),
      { role: 'user', content: message },
    ];

    // Call OpenAI
    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: chatMessages as any,
      temperature: 0.7,
      max_tokens: 2000,
    });

    const response = completion.choices[0].message.content;

    // Save messages to database
    await supabaseClient.from('messages').insert([
      { project_id: projectId, role: 'user', content: message },
      { project_id: projectId, role: 'assistant', content: response },
    ]);

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

Deploy Edge Function:
```bash
# Install Supabase CLI
npm install -g supabase

# Login
supabase login

# Link project
supabase link --project-ref your-project-ref

# Deploy function
supabase functions deploy chat

# Set secrets
supabase secrets set OPENAI_API_KEY=sk-...
```

---

## 🎯 DAY 6: Live Preview (WebContainer)

**File: `src/lib/webcontainer.ts`**

```typescript
import { WebContainer } from '@webcontainer/api';

let webcontainerInstance: WebContainer;

export async function bootWebContainer(): Promise<WebContainer> {
  if (webcontainerInstance) return webcontainerInstance;

  webcontainerInstance = await WebContainer.boot();
  return webcontainerInstance;
}

export async function createViteProject(files: Record<string, string>): Promise<string> {
  const container = await bootWebContainer();

  // Create Vite project structure
  const fileTree: any = {
    'package.json': {
      file: {
        contents: JSON.stringify({
          name: 'preview-app',
          private: true,
          version: '0.0.0',
          type: 'module',
          scripts: {
            dev: 'vite',
            build: 'vite build',
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
          },
        }, null, 2),
      },
    },
    'index.html': {
      file: {
        contents: `<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Preview</title>
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
    src: { directory: {} },
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

  // Install & start
  const installProcess = await container.spawn('npm', ['install']);
  await installProcess.exit;

  const devProcess = await container.spawn('npm', ['run', 'dev']);

  return new Promise((resolve) => {
    container.on('server-ready', (port, url) => {
      resolve(url);
    });
  });
}
```

**File: `src/components/preview/LivePreview.tsx`**

```typescript
import { useEffect, useState } from 'react';
import { usePreviewStore } from '@/stores/preview-store';
import { RefreshCw } from 'lucide-react';

export function LivePreview() {
  const { url, reload } = usePreviewStore();

  return (
    <div className="flex flex-col h-full">
      {/* Toolbar */}
      <div className="flex items-center justify-between p-3 border-b">
        <h2 className="font-semibold">Preview</h2>
        <button
          onClick={reload}
          className="p-2 hover:bg-gray-100 rounded"
        >
          <RefreshCw className="w-4 h-4" />
        </button>
      </div>

      {/* Preview iframe */}
      <div className="flex-1 bg-gray-50 flex items-center justify-center">
        {url ? (
          <iframe
            src={url}
            className="w-full h-full bg-white"
            sandbox="allow-scripts allow-same-origin"
            title="Preview"
          />
        ) : (
          <div className="text-center text-gray-500">
            <p>No preview available</p>
            <p className="text-sm">Start coding to see your app</p>
          </div>
        )}
      </div>
    </div>
  );
}
```

**File: `src/stores/preview-store.ts`**

```typescript
import { create } from 'zustand';

interface PreviewStore {
  url: string;
  setUrl: (url: string) => void;
  reload: () => void;
}

export const usePreviewStore = create<PreviewStore>((set, get) => ({
  url: '',
  setUrl: (url) => set({ url }),
  reload: () => {
    const currentUrl = get().url;
    set({ url: '' });
    setTimeout(() => set({ url: currentUrl }), 100);
  },
}));
```

---

## 🎯 DAY 7: Sidebar & File Management

**File: `src/components/sidebar/Sidebar.tsx`**

```typescript
import { SectionsPanel } from './SectionsPanel';
import { FilesPanel } from './FilesPanel';
import { LayoutGrid, Folder } from 'lucide-react';
import { useState } from 'react';

export function Sidebar() {
  const [tab, setTab] = useState<'sections' | 'files'>('sections');

  return (
    <aside className="w-80 border-r bg-white flex flex-col">
      {/* Tabs */}
      <div className="flex border-b">
        <button
          onClick={() => setTab('sections')}
          className={`flex-1 p-3 flex items-center justify-center gap-2 ${
            tab === 'sections' ? 'bg-blue-50 text-blue-600' : 'text-gray-600'
          }`}
        >
          <LayoutGrid className="w-4 h-4" />
          <span>Sections</span>
        </button>
        <button
          onClick={() => setTab('files')}
          className={`flex-1 p-3 flex items-center justify-center gap-2 ${
            tab === 'files' ? 'bg-blue-50 text-blue-600' : 'text-gray-600'
          }`}
        >
          <Folder className="w-4 h-4" />
          <span>Files</span>
        </button>
      </div>

      {/* Content */}
      <div className="flex-1 overflow-y-auto">
        {tab === 'sections' ? <SectionsPanel /> : <FilesPanel />}
      </div>
    </aside>
  );
}
```

**File: `src/components/sidebar/SectionsPanel.tsx`**

```typescript
import { useChatStore } from '@/stores/chat-store';

const sections = [
  { id: 'hero', name: 'Hero Section', prompt: 'Add a hero section with headline and CTA' },
  { id: 'features', name: 'Features Grid', prompt: 'Add a 3-column features section' },
  { id: 'pricing', name: 'Pricing Table', prompt: 'Add pricing cards with 3 tiers' },
  { id: 'cta', name: 'Call to Action', prompt: 'Add a CTA section with button' },
];

export function SectionsPanel() {
  const { sendMessage } = useChatStore();

  return (
    <div className="p-4 space-y-3">
      <h3 className="font-semibold mb-2">Add Section</h3>
      {sections.map((section) => (
        <button
          key={section.id}
          onClick={() => sendMessage(section.prompt)}
          className="w-full p-4 border rounded-lg hover:border-blue-500 hover:bg-blue-50 text-left transition"
        >
          <p className="font-medium">{section.name}</p>
          <p className="text-sm text-gray-500">Click to add</p>
        </button>
      ))}
    </div>
  );
}
```

**File: `src/components/sidebar/FilesPanel.tsx`**

```typescript
import { useProjectStore } from '@/stores/project-store';
import { File, Folder } from 'lucide-react';

export function FilesPanel() {
  const { files } = useProjectStore();

  return (
    <div className="p-4">
      <h3 className="font-semibold mb-3">Project Files</h3>
      <div className="space-y-1">
        {files.map((file) => (
          <div
            key={file.path}
            className="flex items-center gap-2 p-2 hover:bg-gray-100 rounded cursor-pointer"
          >
            <File className="w-4 h-4 text-blue-500" />
            <span className="text-sm">{file.path}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 🎯 DAY 8: Run & Test

### Step 1: Update Vite Config

**File: `vite.config.ts`**

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
    headers: {
      'Cross-Origin-Embedder-Policy': 'require-corp',
      'Cross-Origin-Opener-Policy': 'same-origin',
    },
  },
});
```

### Step 2: Types

**File: `src/types/index.ts`**

```typescript
export interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  created_at: string;
}

export interface Project {
  id: string;
  user_id: string;
  name: string;
  description?: string;
  file_tree: Record<string, any>;
  created_at: string;
  updated_at: string;
}

export interface ProjectFile {
  id: string;
  project_id: string;
  path: string;
  content: string;
  language?: string;
}
```

### Step 3: Project Store

**File: `src/stores/project-store.ts`**

```typescript
import { create } from 'zustand';
import { ProjectFile } from '@/types';

interface ProjectStore {
  files: ProjectFile[];
  addFile: (file: ProjectFile) => void;
  updateFile: (path: string, content: string) => void;
}

export const useProjectStore = create<ProjectStore>((set) => ({
  files: [],
  addFile: (file) => set((state) => ({ files: [...state.files, file] })),
  updateFile: (path, content) =>
    set((state) => ({
      files: state.files.map((f) =>
        f.path === path ? { ...f, content } : f
      ),
    })),
}));
```

### Step 4: Run Application

```bash
# Start development server
npm run dev

# Open http://localhost:3000
```

### Step 5: Test Flow

1. **Chat**: Type "Create a landing page with hero section"
2. **AI Response**: Should get code suggestions
3. **Preview**: Should see live preview (once WebContainer is integrated)
4. **Files**: Check files panel for generated code

---

## 🎯 Next Steps

### Week 2: Advanced Features
- Code editor with Monaco
- File tree with drag & drop
- Real-time collaboration
- GitHub integration

### Week 3: Production
- User authentication
- Project saving/loading
- Deployment to Vercel
- Custom domains

---

## 📝 Troubleshooting

### Issue 1: WebContainer not loading
```typescript
// Add COOP/COEP headers in vite.config.ts
server: {
  headers: {
    'Cross-Origin-Embedder-Policy': 'require-corp',
    'Cross-Origin-Opener-Policy': 'same-origin',
  },
}
```

### Issue 2: Supabase Edge Function errors
```bash
# Check logs
supabase functions logs chat

# Test locally
supabase functions serve chat
```

### Issue 3: CORS errors
```typescript
// In Edge Function, ensure CORS headers are set
const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};
```

---

## ✅ MVP Checklist

- [ ] Vite project created
- [ ] Supabase setup complete
- [ ] Database schema deployed
- [ ] Chat interface working
- [ ] Edge Function deployed
- [ ] WebContainer integrated
- [ ] Live preview working
- [ ] Sidebar sections clickable
- [ ] Files panel shows generated code
- [ ] App deployed to Vercel

---

**Bây giờ bạn có working Lovable Clone với ĐÚNG tech stack! 🚀**
