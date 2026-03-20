---
name: civic-developer-guide-nextjs-better-auth-vercelai
description: Integration guide for building Next.js apps with Better Auth authentication, Civic MCP tool calling, and Vercel AI SDK. Use when setting up auth, configuring Civic token exchange, wiring up MCP tools, or debugging integration issues.
---

# Civic + Better Auth + Vercel AI SDK — Developer Integration Guide

This skill contains everything needed to build a Next.js app that uses Better Auth for authentication, Civic for AI tool calling (via MCP), and the Vercel AI SDK with Anthropic as the LLM provider.

## Architecture Overview

The integration has three layers:

1. **Better Auth** handles user authentication (email/password, social logins, etc.) and issues RS256-signed JWTs via its built-in JWT plugin.
2. **Civic MCP Client** (`@civic/mcp-client`) exchanges the Better Auth JWT for a Civic access token via RFC 8693 token exchange, then connects to Civic's MCP hub to expose tools (Gmail, Calendar, etc.).
3. **Vercel AI SDK** (`ai`) uses those tools with an Anthropic LLM to build an agent.

The flow: User logs in → Better Auth issues JWT → Civic exchanges JWT for access token → MCP client fetches tools → AI agent uses tools.

### Better Auth vs Auth.js — Key Differences

Better Auth simplifies the integration significantly compared to Auth.js:
- **Automatic RSA key generation** — the JWT plugin generates and stores RS256 key pairs in the database. No manual `openssl` key generation or env var configuration needed.
- **Built-in JWKS endpoint** — automatically available at `/api/auth/jwks`. No custom route needed.
- **Built-in token endpoint** — `/api/auth/token` issues JWTs for token exchange. No custom JWT encode/decode logic needed.
- **SQLite database** — Better Auth persists users and sessions in a real database (SQLite for dev, any SQL DB for production).

## Key Resources

### Documentation
- Civic app integration overview: https://docs.civic.com/civic/developers/integration/apps
- Auth.js / Better Auth / third-party provider setup: https://docs.civic.com/civic/developers/integration/apps#auth-js-%2F-better-auth-%2F-google-%2F-other
- Civic MCP concepts: https://docs.civic.com/labs/concepts/mcp
- Vercel AI SDK tools & tool calling: https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling
- Vercel AI SDK building agents: https://ai-sdk.dev/docs/agents/building-agents
- Vercel AI SDK Anthropic provider: https://ai-sdk.dev/providers/ai-sdk-providers/anthropic
- Better Auth documentation: https://www.better-auth.com/docs

### Reference Implementation
- Full working demo (Better Auth + Civic MCP): https://github.com/civicteam/demos/tree/main/apps/better-auth-demo
- Simplified version: https://github.com/civicteam/demos/tree/main/simple-app-better-auth

### NPM Packages
- `@civic/mcp-client` — Civic MCP client with token exchange and SDK adapters
- `@civic/mcp-client/adapters/vercel-ai` — Vercel AI SDK adapter
- `ai` — Vercel AI SDK core
- `@ai-sdk/react` — React hooks (`useChat`)
- `@ai-sdk/anthropic` — Anthropic provider
- `@ai-sdk/mcp` — Required peer dependency of `@civic/mcp-client`
- `better-auth` — Better Auth framework
- `better-sqlite3` — SQLite driver for Better Auth (dev/demo use)
- `jose` — JWT key conversion (for public key export)

## Environment Variables

All required env vars (see `.env.example` for a template):

```
# Better Auth
BETTER_AUTH_SECRET=<generate with: openssl rand -base64 32>
BETTER_AUTH_URL=https://localhost:<port>

# Civic Auth (from auth.civic.com dashboard)
CIVIC_CLIENT_ID=
CIVIC_CLIENT_SECRET=

# LLM
ANTHROPIC_API_KEY=
```

Note: Unlike Auth.js, Better Auth does **not** require manual RSA key env vars (`JWT_PUBLIC_KEY`, `JWT_PRIVATE_KEY`) or an explicit `ISSUER`/`AUDIENCE`. The JWT plugin handles key generation and the issuer is derived from `BETTER_AUTH_URL`.

## Critical Gotchas

### 1. Self-signed cert causes `fetch` failures on localhost

When using HTTPS locally with self-signed certs (via `mkcert`), server-side `fetch()` calls to your own endpoints (e.g. `/api/auth/get-session`) will fail with `UNABLE_TO_VERIFY_LEAF_CERTIFICATE`.

**Fix:** Call Better Auth's handler directly instead of fetching over HTTPS:

```typescript
import { auth } from "@/lib/auth/server";
import { headers } from "next/headers";

const callAuthEndpoint = async (path: string): Promise<Response> => {
  const headersList = await headers();
  const cookie = headersList.get("cookie") || "";
  return auth.handler(
    new Request(`https://localhost${path}`, { headers: { cookie } }),
  );
};
```

This avoids network calls entirely. Use this pattern for `get-session`, `token`, and `jwks` endpoints.

### 2. getServerInstructions() requires getTools() first

`CivicMcpClient.getServerInstructions()` throws if called before `getTools()` with an adapter, because `getTools()` initializes the MCP connection. Always call them sequentially:

```typescript
// CORRECT — sequential
const tools = await getTools();
const instructions = await getServerInstructions();

// WRONG — parallel (getServerInstructions will throw)
const [tools, instructions] = await Promise.all([getTools(), getServerInstructions()]);
```

### 3. Vercel AI SDK v6 useChat API changes

In `ai@6.x` / `@ai-sdk/react@3.x`, `useChat` no longer exposes `input`/`setInput`. You must:
- Manage input state yourself with `useState`
- Use `DefaultChatTransport` for the API endpoint (not an `api` option)
- Use `sendMessage({ text: input })` instead of `handleSubmit`

### 4. Session cookie name varies by protocol

Better Auth uses different cookie names depending on the protocol:
- `better-auth.session_token` — for http (local dev)
- `__Secure-better-auth.session_token` — for https

Always check both in middleware:

```typescript
const sessionCookie =
  request.cookies.get("better-auth.session_token") ||
  request.cookies.get("__Secure-better-auth.session_token");
```

### 5. Public key for Civic dashboard (local dev)

Civic Auth's servers can't reach localhost to verify JWTs via your JWKS endpoint. For local dev, export the public key in PEM format and paste it into the Civic dashboard.

Create a `/api/keys/public` route that calls the auth handler directly:

```typescript
import { auth } from "@/lib/auth/server";
import { exportSPKI, importJWK } from "jose";

export const GET = async () => {
  const jwksRes = await auth.handler(new Request("https://localhost/api/auth/jwks"));
  const jwks = await jwksRes.json();
  const cryptoKey = await importJWK(jwks.keys[0], "RS256");
  const pem = await exportSPKI(cryptoKey as CryptoKey);
  return new Response(pem, { headers: { "Content-Type": "text/plain" } });
};
```

Visit this endpoint and paste the PEM key into the Civic dashboard (Settings → Integration → Static public key).

### 6. Better Auth password minimum length

Better Auth requires passwords of at least 8 characters by default. If using prefilled demo credentials, ensure the password meets this requirement.

### 7. Database migration required

Before first run, initialize the Better Auth database:

```bash
npx @better-auth/cli migrate --config ./app/lib/auth/server.ts -y
```

## Step-by-Step Integration

### Step 1: Better Auth Server Configuration

Configure Better Auth with the JWT and OIDC Provider plugins. The JWT plugin auto-generates RS256 keys and the OIDC provider plugin exposes a `/api/auth/token` endpoint for obtaining JWTs.

Key file: `app/lib/auth/server.ts`

```typescript
import { betterAuth } from "better-auth";
import { jwt, oidcProvider } from "better-auth/plugins";
import Database from "better-sqlite3";

export const auth: any = betterAuth({
  database: new Database("./auth.db"),
  emailAndPassword: { enabled: true },
  plugins: [
    jwt({ jwks: { keyPairConfig: { alg: "RS256" } } }),
    oidcProvider({ loginPage: "/", consentPage: "/", useJWTPlugin: true }),
  ],
});
```

### Step 2: Better Auth Client

Create the client-side auth hooks with the JWT client plugin:

```typescript
"use client";
import { createAuthClient } from "better-auth/react";
import { jwtClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  plugins: [jwtClient()],
});

export const { signIn, signUp, signOut, useSession } = authClient;
```

### Step 3: Auth API Route

Expose Better Auth's handlers via a catch-all route:

```typescript
import { auth } from "@/lib/auth/server";
import { toNextJsHandler } from "better-auth/next-js";

export const { POST, GET } = toNextJsHandler(auth);
```

### Step 4: Civic Dashboard Configuration

In the Civic dashboard (auth.civic.com), under Settings → Integration:

1. Set the **Identity Provider** type
2. Configure the **Issuer URL** — must match `BETTER_AUTH_URL` (e.g. `https://localhost:3800`)
3. Start the app, visit `/api/keys/public`, and paste the PEM public key into the **Static public key** field
4. Note the **Client ID** and **Client Secret** for your `.env`

### Step 5: Civic MCP Client

The MCP client handles token exchange and tool fetching. The `subjectToken` function provides the Better Auth JWT, and `CivicMcpClient` handles the RFC 8693 exchange internally.

Key file: `app/lib/ai/mcp.ts`

The key pattern is:
1. Use `auth.handler()` to call Better Auth's `/api/auth/token` endpoint directly (avoids self-signed cert issues — see Gotcha #1)
2. Pass the token function as `subjectToken` to `CivicMcpClient` — the client handles RFC 8693 exchange internally
3. Cache clients per user to avoid reconnecting on every request

```typescript
const client = new CivicMcpClient({
  auth: {
    tokenExchange: {
      clientId: process.env.CIVIC_CLIENT_ID!,
      clientSecret: process.env.CIVIC_CLIENT_SECRET!,
      subjectToken: getBetterAuthToken, // returns JWT from auth.handler("/api/auth/token")
    },
  },
});

const tools = await client.getTools(vercelAIAdapter());
```

Reference: https://github.com/civicteam/demos/blob/main/simple-app-better-auth/app/lib/ai/mcp.ts

### Step 6: AI Agent API Route

Create a `POST /api/chat` route that calls `getTools()` then `getServerInstructions()` (sequentially — see Gotcha #2), appends server instructions to the system prompt, and streams the response via `streamText()` with `stopWhen: stepCountIs(15)`. Return `result.toUIMessageStreamResponse()`.

Reference: https://github.com/civicteam/demos/blob/main/simple-app-better-auth/app/api/chat/route.ts

### Step 7: Frontend with useChat

Use `useChat` from `@ai-sdk/react` with `DefaultChatTransport` from `ai`. Manage input state with `useState` and send via `sendMessage({ text: input })`. See Gotcha #3 for details on the v6 API changes.

## Middleware Pattern

Use Next.js middleware to protect routes by checking for the Better Auth session cookie. Redirect unauthenticated users away from protected routes and authenticated users away from the login page.

Key points:
- Check both `better-auth.session_token` and `__Secure-better-auth.session_token` cookies (see Gotcha #4)
- Exclude `api` routes from the matcher so Better Auth endpoints and the chat endpoint work
- Exclude `_next`, static assets from the matcher

## Debugging Token Exchange Errors

If you see `invalid_grant` from Civic:

1. **Check BETTER_AUTH_URL uses https** — Civic Auth rejects `http://` issuers
2. **Verify the issuer matches** — the `iss` claim in the JWT must exactly match what's configured in the Civic dashboard (no trailing slashes, same protocol)
3. **Check the public key** — the key in the Civic dashboard must match the key from your `/api/keys/public` endpoint
4. **Regenerate keys if needed** — delete `auth.db` and restart the app to regenerate RSA keys, then re-export the public key
5. **Check the JWT is valid** — add `console.log("Token:", await getBetterAuthToken())` and decode it at jwt.io to verify claims

## Available Civic MCP Tools

Once connected, the tools available depend on what the user has configured in their Civic profile. Common tool categories include:
- **Gmail** — read, search, send, draft emails
- **Google Calendar** — read events, create events, check availability
- **Jira** — read/create issues
- **Confluence** — read/create pages

The tools are dynamically provided by Civic's MCP hub based on the user's connected services and profile configuration. Use `getServerInstructions()` to get Civic's guidance on how to use the available tools — include this in your system prompt.

## Package Version Compatibility

Tested working combination (as of 2026-03):

```json
{
  "@ai-sdk/anthropic": "^3.0.0",
  "@ai-sdk/mcp": "^1.0.19",
  "@ai-sdk/react": "^3.0.1",
  "@civic/mcp-client": "^0.1.0",
  "ai": "^6.0.79",
  "better-auth": "^1.2.0",
  "better-sqlite3": "^11.0.0",
  "jose": "^6.0.11",
  "next": "^16.1.5"
}
```

`@ai-sdk/mcp` is a required peer dependency of `@civic/mcp-client` — install it even though you don't import it directly.

## HTTPS Setup for Local Development

Better Auth needs HTTPS for secure session cookies. Use [mkcert](https://github.com/FiloSottile/mkcert) to generate trusted local certificates (`brew install mkcert && mkcert -install`), then generate certs and pass them to `next dev` via `--experimental-https-key` and `--experimental-https-cert` flags.

Note: Even after `mkcert -install`, Node.js server-side `fetch()` may not trust the local CA. Use the `auth.handler()` direct-call pattern (see Gotcha #1) to avoid this.
