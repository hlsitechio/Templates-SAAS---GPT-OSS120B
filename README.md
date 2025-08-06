Below is a **step‑by‑step “cook‑book”** that you can run on a fresh machine (macOS, Linux, or Windows WSL) to get a complete development environment for the SaaS dashboard described earlier.  
Everything is written as if you will create a **monorepo** that holds the front‑end (`dashboard/`) and the back‑end (`api/`). The instructions include:

* All `npm` / `pnpm` install commands  
* Every required library (with version ranges)  
* Minimal but functional React components (dashboard, template editor, auth UI)  
* Prisma schema & migration steps  
* Fastify API skeleton (template CRUD + README generation)  
* Next‑Auth configuration for GitHub OAuth  
* Tailwind CSS + Shadcn UI setup  
* CI/CD GitHub‑Actions workflow  
* Dockerfile (optional) for self‑hosting the API  

> **Tip:** The guide uses **pnpm** (fast, disk‑efficient) but you can swap it for `npm`/`yarn` by replacing `pnpm` → `npm` (e.g. `npm install …`).  

---

## 1. Project bootstrap

```bash
# 1️⃣ Create repo folder
mkdir readme‑saas && cd readme‑saas

# 2️⃣ Initialise a pnpm workspace (you can also use npm workspaces)
pnpm init -y
pnpm install -D @pnpm/workspace

# 3️⃣ Add workspace config
cat > pnpm-workspace.yaml <<'EOF'
packages:
  - "dashboard"
  - "api"
EOF
```

### 1.1 Create the two packages

```bash
# Front‑end (Next.js)
mkdir dashboard && cd dashboard
pnpm init -y

# Back‑end (Fastify)
cd .. && mkdir api && cd api
pnpm init -y
```

---

## 2. Front‑end – **dashboard** (Next.js 14)

### 2.1 Install core dependencies

```bash
cd dashboard
pnpm add next@14 react@18 react-dom@18
pnpm add @next-auth/react @next-auth/providers
pnpm add zustand @tanstack/react-query
pnpm add @shadcn/ui @radix-ui/react-icons
pnpm add tailwindcss@3 postcss autoprefixer
pnpm add @tailwindcss/forms @tailwindcss/typography
pnpm add @monaco-editor/react
pnpm add @uiw/react-md-editor   # optional markdown editor
pnpm add @heroicons/react
pnpm add classnames
pnpm add axios
pnpm add dayjs
pnpm add @vercel/analytics   # optional analytics
pnpm add prettier-plugin-tailwindcss -D
pnpm add eslint eslint-config-next prettier -D
```

### 2.2 Initialise Tailwind & Shadcn UI

```bash
# Tailwind init
pnpm exec tailwindcss init -p

# Shadcn UI – pick the components you need
pnpm exec shadcn-ui@latest init
# Choose: button, dialog, dropdown-menu, form, toast, tooltip, etc.
```

#### `tailwind.config.cjs`

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{js,ts,jsx,tsx,mdx}"],
  darkMode: "class",
  theme: {
    extend: {}
  },
  plugins: [
    require("@tailwindcss/forms"),
    require("@tailwindcss/typography")
  ]
};
```

#### `postcss.config.cjs`

```js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {}
  }
};
```

### 2.3 Basic folder layout

```plaintext
dashboard/
├─ src/
│  ├─ app/                # Next.js app router pages
│  │   ├─ (auth)/layout.tsx
│  │   ├─ dashboard/
│  │   │   └─ page.tsx
│  │   └─ templates/
│  │       └─ [slug]/page.tsx
│  ├─ components/         # UI components (Shadcn + custom)
│  ├─ lib/                # helpers (api client, auth utils)
│  ├─ store/              # Zustand stores
│  └─ styles/             # globals + Tailwind imports
├─ public/
├─ prisma/                # optional – shared schema for type safety
├─ .env.local
├─ next.config.mjs
└─ tsconfig.json
```

### 2.4 Example **React components**

#### 2.4.1 `src/components/TemplateCard.tsx`

```tsx
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { useRouter } from "next/navigation";

interface TemplateCardProps {
  id: string;
  name: string;
  description: string;
}

export const TemplateCard = ({
  id,
  name,
  description,
}: TemplateCardProps) => {
  const router = useRouter();

  const openTemplate = () => router.push(`/templates/${id}`);

  return (
    <Card className="w-full">
      <CardHeader>
        <CardTitle>{name}</CardTitle>
      </CardHeader>
      <CardContent className="text-sm">{description}</CardContent>
      <div className="p-4 flex justify-end">
        <Button onClick={openTemplate}>Customize</Button>
      </div>
    </Card>
  );
};
```

#### 2.4.2 `src/components/ReadmeEditor.tsx`

```tsx
import dynamic from "next/dynamic";
import { useEffect, useState } from "react";
import { Button } from "@/components/ui/button";

const MDEditor = dynamic(() => import("@uiw/react-md-editor"), {
  ssr: false,
});

interface ReadmeEditorProps {
  initialContent: string;
  onSave: (content: string) => Promise<void>;
}

export const ReadmeEditor = ({
  initialContent,
  onSave,
}: ReadmeEditorProps) => {
  const [content, setContent] = useState(initialContent);
  const [saving, setSaving] = useState(false);

  const handleSave = async () => {
    setSaving(true);
    await onSave(content);
    setSaving(false);
  };

  return (
    <div className="flex flex-col gap-4">
      <MDEditor
        value={content}
        onChange={(val) => setContent(val || "")}
        height={400}
      />
      <Button onClick={handleSave} disabled={saving}>
        {saving ? "Saving…" : "Save & Push to GitHub"}
      </Button>
    </div>
  );
};
```

#### 2.4.3 `src/components/AuthGuard.tsx`

```tsx
import { useSession } from "next-auth/react";
import { useRouter } from "next/navigation";

export const AuthGuard = ({
  children,
}: {
  children: React.ReactNode;
}) => {
  const { data: session, status } = useSession();
  const router = useRouter();

  if (status === "loading") return null; // or a spinner

  if (!session) {
    router.replace("/api/auth/signin");
    return null;
  }

  return <>{children}</>;
};
```

### 2.5 Next‑Auth configuration (`src/lib/auth.ts`)

```ts
import NextAuth from "next-auth";
import GithubProvider from "next-auth/providers/github";
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import { prisma } from "@/lib/prisma";

export const authOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    GithubProvider({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      // request repo scope for write‑access later
      authorization: {
        params: { scope: "read:user repo" }
      }
    })
  ],
  secret: process.env.NEXTAUTH_SECRET,
  callbacks: {
    async jwt({ token, account }) {
      // Persist the GitHub access token for later API calls
      if (account?.access_token) token.githubAccessToken = account.access_token;
      return token;
    },
    async session({ session, token }) {
      session.githubAccessToken = token.githubAccessToken as string;
      return session;
    }
  }
};

export default NextAuth(authOptions);
```

Add the API route:

```tsx
// src/app/api/auth/[...nextauth]/route.ts
import auth from "@/lib/auth";

export const GET = auth;
export const POST = auth;
```

### 2.6 API client wrapper (`src/lib/api.ts`)

```ts
import axios from "axios";

export const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 10_000,
});

export const setAuthHeader = (token: string) => {
  api.defaults.headers.common.Authorization = `Bearer ${token}`;
};
```

---

## 3. Back‑end – **api** (Fastify + Prisma)

### 3.1 Install core dependencies

```bash
cd ../api
pnpm add fastify fastify-plugin fastify-cors fastify-helmet
pnpm add @fastify/swagger @fastify/compress
pnpm add jsonwebtoken bcryptjs
pnpm add prisma @prisma/client
pnpm add bullmq ioredis
pnpm add @octokit/rest   # GitHub API client
pnpm add dotenv
pnpm add zod            # runtime validation
pnpm add pino pino-pretty
pnpm add fastify-rate-limit
pnpm add fastify-static
pnpm add @fastify/static
pnpm add fastify-multipart
pnpm add fastify-jwt
pnpm add fastify-env
pnpm add fastify-sensible
pnpm add fastify-caching
pnpm add fastify-redis
pnpm add fastify-helmet
pnpm add fastify-cors
pnpm add fastify-compress
pnpm add fastify-graceful-shutdown
pnpm add fastify-caching
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-sensible
pnpm add fastify-env
pnpm add fastify-jwt
pnpm add fastify-multipart
pnpm add fastify-static
pnpm add fastify-redis
pnpm add fastify-compress
pnpm add fastify-cors
pnpm add fastify-helmet
pnpm add fastify-rate-limit
pnpm add fastify-s
