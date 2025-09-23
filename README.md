# Steps

- [ ] `npx create-next-app@latest . --typescript --eslint --app --src-dir --import-alias "@/*" --yes`. Look into integrating supabase at start with `npx create-next-app -e with-supabase`. This can help with auth too.
- [ ] `npm run dev` and check on [http://localhost:3000](http://localhost:3000)
- [ ] `Ask AI to create all necessary html pages. For CSS, see NSRL CRM project. Be sure to
- [ ] Use BEM.
- [ ] Mobile First.
- [ ] Auth with betterAuth. For now, use supabase. Ask AI for steps. For testing, disable email verification from Supabase UI.
- [ ] Install Supabase `npm install @supabase/supabase-js`
- [ ] Set up the Supabase Client in /lib/supabase.ts
```
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
```
- [ ] Add to .env.local. Get values from Project → Settings → API. Copy "Project URL" for NEXT_PUBLIC_SUPABASE_URL. Copy "anon public" key under "Config" → "API keys" for NEXT_PUBLIC_SUPABASE_ANON_KEY.
```
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```
- [ ] Create signup page at app/(auth)/signup/page.tsx
```
'use client'

import { useState, FormEvent } from 'react'
import { supabase } from '@/lib/supabaseClient'
import { useRouter } from 'next/navigation'

export default function SignUpPage() {
  const router = useRouter()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault()

    const { error } = await supabase.auth.signUp({
      email,
      password,
    })

    if (error) {
      setError(error.message)
    } else {
      router.push('/login')
    }
  }

  return (
    <div style={{ padding: 20 }}>
      <h1>Sign Up</h1>
      <form onSubmit={handleSubmit}>
        <input
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          placeholder="Email"
          required
        /><br /><br />
        <input
          type="password"
          value={password}
          onChange={e => setPassword(e.target.value)}
          placeholder="Password"
          required
        /><br /><br />
        <button type="submit">Sign Up</button>
        {error && <p style={{ color: 'red' }}>{error}</p>}
      </form>
    </div>
  )
}
```
- [ ] Create login page at app/(auth)/login/page.tsx
```
'use client'

import { useState, FormEvent } from 'react'
import { supabase } from '@/lib/supabaseClient'
import { useRouter } from 'next/navigation'

export default function LoginPage() {
  const router = useRouter()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState<string | null>(null)

  const handleLogin = async (e: FormEvent) => {
    e.preventDefault()

    const { error } = await supabase.auth.signInWithPassword({
      email,
      password,
    })

    if (error) {
      setError(error.message)
    } else {
      router.push('/dashboard')
    }
  }

  return (
    <div style={{ padding: 20 }}>
      <h1>Login</h1>
      <form onSubmit={handleLogin}>
        <input
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          placeholder="Email"
          required
        /><br /><br />
        <input
          type="password"
          value={password}
          onChange={e => setPassword(e.target.value)}
          placeholder="Password"
          required
        /><br /><br />
        <button type="submit">Login</button>
        {error && <p style={{ color: 'red' }}>{error}</p>}
      </form>
    </div>
  )
}
```
- [ ] Use AuthProvider `components/AuthProvider.tsx`.
```
'use client'

import { createContext, useContext, useEffect, useState } from 'react'
import { supabase } from '@/lib/supabaseClient'
import type { Session, User } from '@supabase/supabase-js'

interface AuthContextType {
  user: User | null
  session: Session | null
  loading: boolean
  signOut: () => Promise<void>
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [session, setSession] = useState<Session | null>(null)
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const getSession = async () => {
      const { data, error } = await supabase.auth.getSession()
      if (data?.session) {
        setSession(data.session)
        setUser(data.session.user)
      } else {
        setSession(null)
        setUser(null)
      }
      setLoading(false)
    }

    getSession()

    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session)
      setUser(session?.user ?? null)
    })

    return () => {
      subscription.unsubscribe()
    }
  }, [])

  const signOut = async () => {
    await supabase.auth.signOut()
  }

  return (
    <AuthContext.Provider value={{ user, session, loading, signOut }}>
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider')
  }
  return context
}
```
- [ ] Wrap the app with AuthProvider. `app/layout.tsx`
```
import './globals.css'
import { AuthProvider } from '@/components/AuthProvider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <AuthProvider>
          {children}
        </AuthProvider>
      </body>
    </html>
  )
}
```
- [ ] Create ProtectedLayout to Block Unauthorized Users at `app/(protected)/layout.tsx`
```
'use client'

import { useEffect } from 'react'
import { useRouter } from 'next/navigation'
import { useAuth } from '@/components/AuthProvider'

export default function ProtectedLayout({ children }: { children: React.ReactNode }) {
  const { user, loading } = useAuth()
  const router = useRouter()

  useEffect(() => {
    if (!loading && !user) {
      router.replace('/login')
    }
  }, [loading, user, router])

  if (loading) return <p>Loading...</p>

  return <>{children}</>
}

```
- [ ] The structure will look like this:
```
app/
├─ layout.tsx               ← wraps entire app with AuthProvider
├─ (auth)/
│  ├─ login/
│  ├─ signup/
├─ (protected)/             ← all routes here are guarded
│  ├─ layout.tsx            ← ProtectedLayout (redirects if not logged in)
│  ├─ profile/
│  ├─ dashboard/
components/
├─ AuthProvider.tsx         ← global auth context
lib/
├─ supabase.ts        ← your Supabase client

```



