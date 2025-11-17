# 🚀 Lovable Clone - Quick Start Guide

> Hướng dẫn bắt đầu nhanh để build MVP trong 2-4 tuần

---

## 📋 Prerequisites

```bash
# Required
- Node.js >= 18.0.0
- npm hoặc pnpm
- Git
- PostgreSQL (hoặc Supabase account)

# Optional but Recommended
- Docker Desktop
- VS Code
- OpenAI API key hoặc Anthropic API key
```

---

## 🏁 Quick Start (MVP trong 2 tuần)

### **Bước 1: Setup Project Structure**

```bash
# Create monorepo
npx create-turbo@latest lovable-clone
cd lovable-clone

# Structure
lovable-clone/
├── src/
│   ├── components/   # React components
│   ├── lib/          # Utilities & Supabase client
│   ├── stores/       # Zustand stores
│   └── types/        # TypeScript types
├── supabase/
│   ├── functions/    # Edge Functions
│   └── migrations/   # Database schemas
├── public/
├── index.html
├── vite.config.ts
└── package.json
```

### **Bước 2: Initialize Frontend**

```bash
# Create Vite + React app
npm create vite@latest lovable-clone -- --template react-ts
cd lovable-clone

# Install dependencies
npm install @radix-ui/react-dialog @radix-ui/react-tabs
npm install class-variance-authority clsx tailwind-merge
npm install lucide-react
npm install zustand
npm install react-markdown
npm install @supabase/supabase-js

# Install shadcn/ui
npx shadcn-ui@latest init
npx shadcn-ui@latest add button dialog input textarea tabs scroll-area
```

**File: `src/App.tsx`**
```typescript
import { ChatPanel } from '@/components/chat-panel';
import { LivePreview } from '@/components/live-preview';
import { Sidebar } from '@/components/sidebar';

export default function App() {
  return (
    <div className="flex h-screen bg-background">
      {/* Sidebar */}
      <Sidebar />

      {/* Main Content */}
      <div className="flex-1 flex">
        {/* Chat Panel */}
        <div className="w-1/2 border-r">
          <ChatPanel />
        </div>

        {/* Live Preview */}
        <div className="w-1/2">
          <LivePreview />
        </div>
      </div>
    </div>
  );
}
```

### **Bước 3: Setup Supabase Backend**

```bash
# Install Supabase CLI
npm install -g supabase

# Initialize Supabase in project
supabase init

# Link to Supabase project (or create new)
supabase login
supabase link --project-ref your-project-ref

# Setup database migrations
supabase db push

# Create Edge Functions
supabase functions new chat
supabase functions new codegen
```

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

    // AI logic here...
    const openai = new OpenAI({
      apiKey: Deno.env.get('OPENAI_API_KEY'),
    });

    // ... rest of chat logic

    return new Response(JSON.stringify({ response: 'AI response' }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      status: 400,
    });
  }
});
```

### **Bước 4: Setup Database**

**Option A: Supabase (Recommended - Easier)**

```bash
# Install Supabase client
npm install @supabase/supabase-js

# Go to https://supabase.com
# Create new project
# Copy URL and anon key to .env
```

**File: `.env`**
```env
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJxxx...
```

**Option B: Local PostgreSQL + Prisma**

```bash
cd packages/database

npm install prisma @prisma/client
npx prisma init

# Edit prisma/schema.prisma (copy from ARCHITECTURE.md section VII)
npx prisma migrate dev --name init
npx prisma generate
```

### **Bước 5: Implement Basic Chat**

**File: `src/components/chat-panel.tsx`**
```typescript
import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { ScrollArea } from '@/components/ui/scroll-area';
import { supabase } from '@/lib/supabase';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

export function ChatPanel() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const sendMessage = async () => {
    if (!input.trim()) return;

    const userMessage: Message = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);

    try {
      // Call Supabase Edge Function
      const { data, error } = await supabase.functions.invoke('chat', {
        body: { message: input, history: messages }
      });

      if (error) throw error;

      const assistantMessage: Message = {
        role: 'assistant',
        content: data.response
      };

      setMessages(prev => [...prev, assistantMessage]);
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="flex flex-col h-full">
      {/* Header */}
      <div className="p-4 border-b">
        <h1 className="text-xl font-bold">Chat</h1>
      </div>

      {/* Messages */}
      <ScrollArea className="flex-1 p-4">
        {messages.map((msg, i) => (
          <div
            key={i}
            className={`mb-4 ${msg.role === 'user' ? 'text-right' : 'text-left'}`}
          >
            <div
              className={`inline-block p-3 rounded-lg ${
                msg.role === 'user'
                  ? 'bg-blue-500 text-white'
                  : 'bg-gray-200 text-black'
              }`}
            >
              {msg.content}
            </div>
          </div>
        ))}
      </ScrollArea>

      {/* Input */}
      <div className="p-4 border-t">
        <div className="flex gap-2">
          <Textarea
            value={input}
            onChange={(e) => setInput(e.target.value)}
            placeholder="Describe your app..."
            className="flex-1"
            onKeyDown={(e) => {
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                sendMessage();
              }
            }}
          />
          <Button onClick={sendMessage} disabled={isLoading}>
            {isLoading ? 'Sending...' : 'Send'}
          </Button>
        </div>
      </div>
    </div>
  );
}
```

### **Bước 6: Setup Supabase Client**

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

### **Bước 7: Deploy Edge Function for Code Generation**

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

    const { data: { user } } = await supabaseClient.auth.getUser();
    if (!user) throw new Error('Not authenticated');

    const { requirement, projectId } = await req.json();

    const openai = new OpenAI({
      apiKey: Deno.env.get('OPENAI_API_KEY'),
    });

    // Load Lovable system prompt
    const LOVABLE_PROMPT = await Deno.readTextFile('./prompts/lovable-system.txt');

    const completion = await openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: [
        { role: 'system', content: LOVABLE_PROMPT },
        { role: 'user', content: `Generate a React component for: ${requirement}` }
      ],
      temperature: 0.2,
      max_tokens: 4000
    });

    const code = completion.choices[0].message.content;

    // Extract code from markdown
    const codeMatch = code?.match(/```(?:typescript|tsx)?\n([\s\S]*?)```/);
    const extractedCode = codeMatch ? codeMatch[1] : code;

    return new Response(
      JSON.stringify({ code: extractedCode, success: true }),
      {
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
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

### **Bước 8: Copy Lovable Prompt**

```bash
# Create prompts directory
mkdir -p apps/api/src/prompts

# Copy Lovable agent prompt
cp Lovable/Agent\ Prompt.txt apps/api/src/prompts/lovable-agent.txt
```

### **Bước 9: Setup Environment Variables**

**File: `.env`** (For Edge Functions - set via Supabase CLI)
```bash
# Deploy secrets to Supabase
supabase secrets set OPENAI_API_KEY=sk-...
supabase secrets set ANTHROPIC_API_KEY=sk-ant-...
```

**File: `.env.local`** (For Frontend)
```env
VITE_SUPABASE_URL=https://xxxxx.supabase.co
VITE_SUPABASE_ANON_KEY=eyJxxx...
```

### **Bước 10: Run the App**

```bash
# Terminal 1: Start Supabase locally (optional)
supabase start

# Terminal 2: Run Edge Functions locally (optional)
supabase functions serve

# Terminal 3: Run Vite dev server
npm run dev

# Open http://localhost:5173
```

---

## 🎨 **Bước 11: Implement Live Preview (Basic)**

**File: `src/components/live-preview.tsx`**
```typescript
import { useEffect, useState } from 'react';

export function LivePreview() {
  const [url, setUrl] = useState('');

  useEffect(() => {
    // In production, this would be WebContainer URL
    // For MVP, use static preview
    setUrl('http://localhost:5173');
  }, []);

  return (
    <div className="flex flex-col h-full">
      {/* Toolbar */}
      <div className="p-4 border-b flex items-center gap-4">
        <h2 className="font-bold">Preview</h2>
        <div className="flex gap-2">
          <button className="px-3 py-1 border rounded">Mobile</button>
          <button className="px-3 py-1 border rounded">Tablet</button>
          <button className="px-3 py-1 border rounded bg-blue-500 text-white">
            Desktop
          </button>
        </div>
      </div>

      {/* Preview iframe */}
      <div className="flex-1 bg-gray-100 flex items-center justify-center">
        <div className="w-full h-full bg-white">
          {url ? (
            <iframe
              src={url}
              className="w-full h-full"
              sandbox="allow-scripts allow-same-origin"
            />
          ) : (
            <div className="flex items-center justify-center h-full">
              <p className="text-gray-500">No preview available</p>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

---

## 🔧 **Bước 12: Implement Sidebar (Basic)**

**File: `src/components/sidebar.tsx`**
```typescript

import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs';
import { LayoutGrid, Package, Palette, Folder } from 'lucide-react';

export function Sidebar() {
  return (
    <aside className="w-80 border-r">
      <Tabs defaultValue="sections" className="h-full">
        <TabsList className="w-full justify-start border-b rounded-none">
          <TabsTrigger value="sections">
            <LayoutGrid className="w-4 h-4 mr-2" />
            Sections
          </TabsTrigger>
          <TabsTrigger value="theme">
            <Palette className="w-4 h-4 mr-2" />
            Theme
          </TabsTrigger>
        </TabsList>

        <TabsContent value="sections" className="p-4 space-y-4">
          <h3 className="font-bold mb-2">Add Section</h3>

          {['Hero', 'Features', 'Testimonials', 'CTA', 'Pricing'].map((section) => (
            <button
              key={section}
              className="w-full p-4 border rounded-lg hover:border-blue-500 transition"
            >
              <p className="font-medium">{section}</p>
              <p className="text-sm text-gray-500">Click to add</p>
            </button>
          ))}
        </TabsContent>

        <TabsContent value="theme" className="p-4">
          <h3 className="font-bold mb-4">Theme Customization</h3>

          <div className="space-y-4">
            <div>
              <label className="text-sm font-medium">Primary Color</label>
              <input
                type="color"
                defaultValue="#3b82f6"
                className="w-full h-10 mt-2"
              />
            </div>

            <div>
              <label className="text-sm font-medium">Font Family</label>
              <select className="w-full p-2 border rounded mt-2">
                <option>Inter</option>
                <option>Roboto</option>
                <option>Poppins</option>
              </select>
            </div>
          </div>
        </TabsContent>
      </Tabs>
    </aside>
  );
}
```

---

## 📦 **Bước 13: Add WebContainer (Advanced)**

```bash
npm install @webcontainer/api
```

**File: `src/lib/webcontainer.ts`**
```typescript
import { WebContainer } from '@webcontainer/api';

let webcontainerInstance: WebContainer;

export async function initWebContainer() {
  if (webcontainerInstance) return webcontainerInstance;

  webcontainerInstance = await WebContainer.boot();
  return webcontainerInstance;
}

export async function createProject(files: Record<string, any>) {
  const container = await initWebContainer();

  // Mount files
  await container.mount(files);

  // Install dependencies
  const installProcess = await container.spawn('npm', ['install']);
  const installExitCode = await installProcess.exit;

  if (installExitCode !== 0) {
    throw new Error('Failed to install dependencies');
  }

  // Start dev server
  const devProcess = await container.spawn('npm', ['run', 'dev']);

  // Wait for server to be ready
  devProcess.output.pipeTo(
    new WritableStream({
      write(data) {
        console.log(data);
      }
    })
  );

  // Get preview URL
  container.on('server-ready', (port, url) => {
    console.log('Server ready at:', url);
  });

  return container;
}
```

---

## 🧪 **Testing Your MVP**

### Test Flow:
1. **Chat**: Type "Create a landing page with a hero section"
2. **Generate**: AI should respond with code
3. **Preview**: Should see live preview (if WebContainer is setup)
4. **Iterate**: Ask "Make the hero background blue"
5. **Export**: Add export functionality

---

## 📈 **MVP Features Checklist**

### Week 1:
- [x] Basic chat interface
- [x] AI integration (OpenAI/Anthropic)
- [x] Simple code generation
- [x] Static preview
- [x] Sidebar with sections

### Week 2:
- [ ] WebContainer integration
- [ ] Live code preview
- [ ] File system management
- [ ] Error detection
- [ ] Export to ZIP

### Week 3-4 (Optional):
- [ ] Design system customization
- [ ] Component library
- [ ] GitHub integration
- [ ] Deploy to Vercel
- [ ] User authentication

---

## 🐛 Common Issues & Solutions

### Issue 1: CORS errors
```typescript
// apps/api/src/index.ts
app.use(cors({
  origin: ['http://localhost:3000'],
  credentials: true
}));
```

### Issue 2: OpenAI API rate limits
```typescript
// Add retry logic
import { retry } from '@/lib/retry';

const completion = await retry(() =>
  openai.chat.completions.create({...}),
  { retries: 3, delay: 1000 }
);
```

### Issue 3: WebContainer not loading
```typescript
// Make sure you're using HTTPS in production
// WebContainer requires secure context
```

---

## 🚀 Next Steps After MVP

1. **Add Authentication** (Supabase Auth / Clerk)
2. **Implement Project Saving** (Database persistence)
3. **Add Collaboration** (WebSocket + Yjs)
4. **Deploy to Production** (Vercel + Railway)
5. **Add Billing** (Stripe integration)
6. **Marketing & Launch** (Product Hunt, Twitter)

---

## 📚 Resources

- **Full Architecture**: See `LOVABLE_CLONE_ARCHITECTURE.md`
- **Lovable Prompt**: `/Lovable/Agent Prompt.txt`
- **Lovable Tools**: `/Lovable/Agent Tools.json`
- **WebContainer Docs**: https://webcontainers.io/guides/quickstart
- **shadcn/ui**: https://ui.shadcn.com
- **Vite**: https://vitejs.dev/guide/
- **Supabase**: https://supabase.com/docs

---

## 💡 Pro Tips

1. **Start Simple**: Build chat first, add complexity later
2. **Use Lovable's Prompt**: Copy their system prompt directly
3. **Test Iteratively**: Deploy early, get feedback
4. **Monitor Costs**: Track OpenAI API usage
5. **Cache Responses**: Use Redis for common queries
6. **Version Control**: Git commit frequently
7. **Documentation**: Document as you build
8. **Community**: Join Discord for help

---

**Good luck building your Lovable clone! 🎉**

Nếu cần help với bất kỳ bước nào, hãy reference back to the full architecture document!
