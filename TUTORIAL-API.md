# Tutorial: Connect Next.js to the Sonora Producciones API

Base URL: `https://sonora-producciones-backend.onrender.com/api`
Docs: `https://sonora-producciones-backend.onrender.com/docs`

Scope: registration, login, list events, display events.

## How the API responds

- Success: `{ "data": ... }` (lists also include `meta`).
- Error: `{ "error": { "code", "message", "details" } }`.
- Customers use the session token from `/auth/signin`. Do NOT use `/auth/login` (that is the admin token).

## Step 1: Environment variable

`.env.local`:

```env
NEXT_PUBLIC_API_URL=https://sonora-producciones-backend.onrender.com/api
```

Restart `npm run dev` afterwards.

## Step 2: Base HTTP client

`src/lib/api.ts`:

```ts
const BASE = process.env.NEXT_PUBLIC_API_URL!;

export class ApiError extends Error {
  constructor(public code: string, message: string, public details?: unknown) {
    super(message);
  }
}

export async function api<T>(path: string, init: RequestInit = {}, token?: string) {
  const res = await fetch(`${BASE}${path}`, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...init.headers,
    },
  });

  if (res.status === 204) return null as T;

  const body = await res.json();
  if (!res.ok) {
    throw new ApiError(body.error?.code, body.error?.message, body.error?.details);
  }
  return body as T; // { data, meta? }
}
```

## Step 3: Types

`src/lib/types.ts`:

```ts
export type User = { id: number; name: string; email: string; createdAt: string };

export type TicketType = {
  id: number; eventId: number; name: string;
  price: number; capacity: number; sold: number; available: number;
};

export type Event = {
  id: number; name: string; description: string;
  date: string; startTime: string; venue: string; city: string;
  imageUrl: string; status: "draft" | "published" | "soldout" | "cancelled";
  featured: boolean; createdAt: string;
  ticketTypes: TicketType[]; minPrice: number;
};

export type PageMeta = { page: number; limit: number; total: number; totalPages: number };
```

## Step 4: Register

`POST /auth/register` with `{ name, email, password }`. Returns 201 and does NOT start a session. Existing email returns 409 `email_taken`.

`src/lib/auth.ts`:

```ts
import { api } from "./api";
import type { User } from "./types";

export const register = (name: string, email: string, password: string) =>
  api<{ data: { user: User } }>("/auth/register", {
    method: "POST",
    body: JSON.stringify({ name, email, password }),
  });
```

## Step 5: Login

`POST /auth/signin` with `{ email, password }`. Returns `data.token` (valid 7 days). Wrong credentials return 401. The token is only delivered here.

```ts
export const signin = (email: string, password: string) =>
  api<{ data: { token: string; tokenType: "Bearer"; user: User } }>("/auth/signin", {
    method: "POST",
    body: JSON.stringify({ email, password }),
  });

export const getMe = (token: string) => api<{ data: User }>("/me", {}, token);

export const signout = (token: string) =>
  api<null>("/auth/signout", { method: "POST" }, token);
```

Store the token after login:

```ts
const { data } = await signin(email, password);
localStorage.setItem("token", data.token);
```

Logout: call `signout(token)` (204) and remove it from `localStorage`.

> `localStorage` is fine for class. For production prefer an `httpOnly` cookie set by a Next Route Handler acting as a proxy (XSS risk otherwise).

### Example login form (Client Component)

`src/app/login/page.tsx`:

```tsx
"use client";
import { useState } from "react";
import { signin } from "@/lib/auth";
import { ApiError } from "@/lib/api";

export default function LoginPage() {
  const [error, setError] = useState("");

  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const f = new FormData(e.currentTarget);
    try {
      const { data } = await signin(String(f.get("email")), String(f.get("password")));
      localStorage.setItem("token", data.token);
      window.location.href = "/events";
    } catch (err) {
      setError(err instanceof ApiError ? err.message : "Unexpected error");
    }
  }

  return (
    <form onSubmit={onSubmit} className="mx-auto flex max-w-sm flex-col gap-3 p-6">
      <input name="email" type="email" placeholder="Email" required className="border p-2" />
      <input name="password" type="password" placeholder="Password" required className="border p-2" />
      <button className="bg-black p-2 text-white">Sign in</button>
      {error && <p className="text-red-600">{error}</p>}
    </form>
  );
}
```

The register form is the same shape, with `name`, `email`, `password` calling `register(...)`, then redirect to `/login`.

## Step 6: List events (public, no token)

`GET /events`. Optional query params: `q`, `status`, `city`, `featured`, `from`, `to`, `sort` (`date`, `-date`, `name`, `-name`, `created`, `-created`), `page`, `limit`.

`src/lib/events.ts`:

```ts
import { api } from "./api";
import type { Event, PageMeta } from "./types";

export const getEvents = (params: Record<string, string> = {}) => {
  const qs = new URLSearchParams(params).toString();
  return api<{ data: Event[]; meta: PageMeta }>(`/events${qs ? `?${qs}` : ""}`);
};

export const getEvent = (id: number | string) =>
  api<{ data: Event }>(`/events/${id}`);
```

## Step 7: Display events

`src/app/events/page.tsx` (Server Component):

```tsx
import Link from "next/link";
import { getEvents } from "@/lib/events";

export default async function EventsPage() {
  const { data: events } = await getEvents({ sort: "date", limit: "12" });

  return (
    <main className="mx-auto max-w-5xl p-6">
      <h1 className="mb-6 text-3xl font-bold">Events</h1>
      <ul className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {events.map((e) => (
          <li key={e.id} className="rounded-lg border p-4">
            <h2 className="font-semibold">{e.name}</h2>
            <p className="text-sm">{e.date} · {e.startTime}</p>
            <p className="text-sm">{e.venue}, {e.city}</p>
            <p className="mt-2">From ${e.minPrice.toLocaleString("es-CL")} CLP</p>
            <Link href={`/events/${e.id}`} className="text-blue-600">View details</Link>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

Detail page, `src/app/events/[id]/page.tsx`. In recent Next versions `params` is a Promise:

```tsx
import { getEvent } from "@/lib/events";

export default async function EventPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const { data: e } = await getEvent(id);

  return (
    <main className="mx-auto max-w-3xl p-6">
      <h1 className="text-3xl font-bold">{e.name}</h1>
      <p>{e.description}</p>
      <ul className="mt-4">
        {e.ticketTypes.map((t) => (
          <li key={t.id}>{t.name}: ${t.price} ({t.available} available)</li>
        ))}
      </ul>
    </main>
  );
}
```

If you use `<Image>` with `imageUrl`, allow the domain in `next.config` under `images.remotePatterns`.

## Gotchas

1. **Render free tier cold start**: the first request after inactivity can take 30-60 s. Show a loading state.
2. **CORS**: browser `fetch` from Client Components needs the backend to allow your origin. Server Components are not affected.
3. **`GET /features`** lists the active optional modules (not needed for this scope).
4. **Next.js version**: `AGENTS.md` warns of breaking changes. Check `node_modules/next/dist/docs/` before writing real code.

## Quick check before class

```bash
curl https://sonora-producciones-backend.onrender.com/api/health
curl https://sonora-producciones-backend.onrender.com/api/events
```

Tip: run these first to wake up the server (cold start).
