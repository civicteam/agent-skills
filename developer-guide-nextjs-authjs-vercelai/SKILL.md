---
name: civic-developer-guide-nextjs-authjs-vercelai
description: Integration guide for building Next.js apps with Auth.js authentication, Civic MCP tool calling, and Vercel AI SDK. Use when setting up auth, configuring Civic token exchange, wiring up MCP tools, or debugging integration issues.
---

# Civic + Auth.js + Vercel AI SDK — Developer Integration Guide

This skill contains everything needed to build a Next.js app that uses Auth.js for authentication, Civic for AI tool calling (via MCP), and the Vercel AI SDK with Anthropic as the LLM provider.

## Architecture Overview

The integration has three layers:

1. **Auth.js** handles user authentication (credentials, Google, GitHub, etc.) and issues RSA-signed JWTs.
2. **Civic MCP Client** (`@civic/mcp-client`) exchanges the Auth.js JWT for a Civic access token via RFC 8693 token exchange, then connects to Civic's MCP hub to expose tools (Gmail, Calendar, etc.).
3. **Vercel AI SDK** (`ai`) uses those tools with an Anthropic LLM to build an agent.

The flow: User logs in → Auth.js issues JWT → Civic exchanges JWT for access token → MCP client fetches tools → AI agent uses tools.

## Key Resources

### Documentation
- Civic app integration overview: https://docs.civic.com/civic/developers/integration/apps
- Auth.js / third-party provider setup: https://docs.civic.com/civic/developers/integration/apps#auth-js-%2F-better-auth-%2F-google-%2F-other
- Civic MCP concepts: https://docs.civic.com/labs/concepts/mcp
- Vercel AI SDK tools & tool calling: https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling
- Vercel AI SDK building agents: https://ai-sdk.dev/docs/agents/building-agents
- Vercel AI SDK Anthropic provider: https://ai-sdk.dev/providers/ai-sdk-providers/anthropic

### Reference Implementation
- Full working demo (Auth.js + Civic MCP): https://github.com/civicteam/demos/tree/main/apps/next-auth-demo
- This project (`simple-app-authjs`) is a simplified version of that demo

### NPM Packages
- `@civic/mcp-client` — Civic MCP client with token exchange and SDK adapters
- `@civic/mcp-client/adapters/vercel-ai` — Vercel AI SDK adapter
- `ai` — Vercel AI SDK core
- `@ai-sdk/react` — React hooks (`useChat`)
- `@ai-sdk/anthropic` — Anthropic provider
- `@ai-sdk/mcp` — Required peer dependency of `@civic/mcp-client`
- `next-auth` — Auth.js for Next.js (use `5.0.0-beta.30` or later)
- `jose` — JWT signing and verification
- `zod` — Schema validation for tool inputs

## Environment Variables

All required env vars (see `.env.example` for a template):

```
# LLM
ANTHROPIC_API_KEY=

# Auth.js
NEXTAUTH_URL=http://localhost:<port>
NEXTAUTH_SECRET=<generate with: openssl rand -base64 32>

# CRITICAL: ISSUER must use https:// even on localhost — Civic Auth rejects http:// issuers
ISSUER=https://localhost:<port>
AUDIENCE=https://localhost:<port>

# RSA keys for JWT signing (generate with: pnpm generate-keys or openssl)
JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
JWT_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"

# Civic Auth (from auth.civic.com dashboard)
CIVIC_CLIENT_ID=
CIVIC_CLIENT_SECRET=
CIVIC_PROFILE_ID=
```

### Generating RSA Keys

Use the helper script or openssl directly:

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem
```

Then paste as single-line values with `\n` for newlines.

## Critical Gotchas

### 1. ISSUER must be https

The `ISSUER` env var (which becomes the `iss` claim in JWTs) **must** use `https://`, even for `localhost`. Civic Auth's token exchange endpoint rejects `http://` issuers. The error message is unhelpful:

```
Token exchange failed (400): {"error":"invalid_grant","error_description":"grant request is invalid"}
```

**Fix:** Set `ISSUER=https://localhost:<port>` in `.env`.

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

See the reference demo's `Chatbot.tsx` for the correct pattern:
https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/components/Chatbot.tsx

### 4. JWKS endpoint must be publicly accessible

For Civic to verify your JWTs, it needs access to either:
- Your JWKS endpoint (e.g. `/api/keys/.well-known/jwks.json`) — requires a public URL
- Or the raw public key pasted directly in the Civic dashboard

For local development, use the raw public key approach in the dashboard (Settings → Integration). For production, expose the JWKS endpoint.

### 5. Session token cookie name varies

Auth.js uses different cookie names depending on the protocol:
- `authjs.session-token` — for http (local dev)
- `__Secure-authjs.session-token` — for https (production)

Always check both when reading the session token for Civic token exchange.

## Project Structure

Reference structure for a working integration:

```
├── auth.ts                              # Auth.js config with custom JWT encode/decode
├── middleware.ts                         # Route protection
├── next.config.ts
├── scripts/generate-keys.ts             # RSA key pair generator
├── app/
│   ├── layout.tsx                       # Root layout
│   ├── providers.tsx                    # SessionProvider wrapper
│   ├── page.tsx                         # Login page
│   ├── assistant/page.tsx               # Protected page
│   ├── lib/
│   │   ├── auth/
│   │   │   ├── jwt.ts                   # Custom JWT encode/decode with RSA signing
│   │   │   ├── keys.ts                  # RSA key loading from env vars
│   │   │   └── users.ts                 # User store (demo: in-memory)
│   │   └── ai/
│   │       ├── mcp.ts                   # CivicMcpClient setup + tool fetching
│   │       └── llm.ts                   # Anthropic model config
│   └── api/
│       ├── auth/[...nextauth]/route.ts  # Auth.js route handlers
│       ├── chat/route.ts                # AI agent streaming endpoint
│       └── keys/.well-known/jwks.json/route.ts  # JWKS endpoint for Civic
```

## Step-by-Step Integration

### Step 1: Auth.js with Custom JWT Signing

Auth.js must issue JWTs signed with your RSA private key (not the default encrypted cookie). This is required for Civic's token exchange to verify the token.

Key file: `auth.ts` — configure with `jwt: { encode: encodeJwt, decode: decodeJwt }`.

Reference implementation:
- `auth.ts`: https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/auth.ts
- `jwt.ts`: https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/lib/auth/jwt.ts
- `keys.ts`: https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/lib/auth/keys.ts

The JWT must include these claims: `iss` (issuer), `aud` (audience), `scope: "openid profile email"`, `iat`, `exp`, plus user identity fields (`email`, `name`, `id`).

### Step 2: JWKS Endpoint

Expose your public key at `/api/keys/.well-known/jwks.json` so Civic can verify JWT signatures.

Reference: https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/api/keys/.well-known/jwks.json/route.ts

Alternatively, paste the raw PEM public key directly in the Civic dashboard (easier for local dev).

### Step 3: Civic Dashboard Configuration

In the Civic dashboard (auth.civic.com), under Settings → Integration:

1. Set the **Identity Provider** type
2. Configure the **Issuer URL** — must exactly match the `ISSUER` env var (including `https://`)
3. Provide either:
   - **JWKS URL**: `<your-public-url>/api/keys/.well-known/jwks.json`
   - **Public Key**: paste the raw PEM public key
4. Note the **Client ID** and **Client Secret** for your `.env`
5. Note the **Profile ID** (UUID) for `CIVIC_PROFILE_ID`

### Step 4: Civic MCP Client

The MCP client handles token exchange and tool fetching. Key pattern:

```typescript
import { CivicMcpClient } from "@civic/mcp-client";
import { vercelAIAdapter } from "@civic/mcp-client/adapters/vercel-ai";

const client = new CivicMcpClient({
  auth: {
    tokenExchange: {
      clientId: process.env.CIVIC_CLIENT_ID,
      clientSecret: process.env.CIVIC_CLIENT_SECRET,
      subjectToken: getSessionToken,  // returns Auth.js JWT from cookies
    },
  },
  civicProfile: process.env.CIVIC_PROFILE_ID,
});

const tools = await client.getTools(vercelAIAdapter());
```

The `subjectToken` function reads the Auth.js session cookie. Reference:
https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/lib/ai/mcp.ts

Important: Cache the `CivicMcpClient` per user to avoid reconnecting on every request. The reference implementation uses a `Map<string, CachedClient>` with inactivity-based cleanup.

### Step 5: AI Agent API Route

Wire tools into a `streamText` call with the Vercel AI SDK:

```typescript
import { streamText, stepCountIs } from "ai";

const tools = await getTools();                    // from Step 4
const serverInstructions = await getServerInstructions(); // AFTER getTools

const result = streamText({
  model,
  messages,
  system: `Your system prompt\n\n${serverInstructions}`,
  tools,
  stopWhen: stepCountIs(15),
});

return result.toUIMessageStreamResponse();
```

Reference: https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/api/chat/route.ts

### Step 6: Frontend with useChat

Use the `useChat` hook with `DefaultChatTransport`:

```typescript
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

const [input, setInput] = useState("");
const { messages, sendMessage, status } = useChat({
  transport: new DefaultChatTransport({ api: "/api/chat" }),
});

// Send: sendMessage({ text: input })
```

Reference: https://github.com/civicteam/demos/blob/main/apps/next-auth-demo/app/components/Chatbot.tsx

## Middleware Pattern

Protect routes and redirect authenticated users:

```typescript
import { auth } from "./auth";
import { NextResponse } from "next/server";

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isProtected = req.nextUrl.pathname.startsWith("/assistant");
  if (isProtected && !isLoggedIn) return NextResponse.redirect(new URL("/", req.nextUrl));
  if (isLoggedIn && req.nextUrl.pathname === "/") return NextResponse.redirect(new URL("/assistant", req.nextUrl));
  return NextResponse.next();
});

export const config = {
  matcher: ["/((?!_next|favicon.ico|api|.*\\.jpg|.*\\.png|.*\\.svg|.*\\.gif).*)"],
};
```

Note: The `api` routes must be excluded from the matcher so Auth.js callback routes and the chat endpoint work.

## Debugging Token Exchange Errors

If you see `invalid_grant` from Civic:

1. **Check ISSUER is https** — most common cause
2. **Verify the `iss` claim in the JWT matches** what's configured in the Civic dashboard exactly (no trailing slashes, same protocol)
3. **Check the public key / JWKS** — the key Civic has must match the private key signing the JWTs
4. **Check the JWT has required claims** — `iss`, `aud`, `exp`, `iat`, `scope`
5. **Log the session token** to verify it's a real JWT (not an encrypted cookie): add `console.log("Session token:", sessionToken)` in `getSessionToken()`
6. **Verify cookie names** — check both `authjs.session-token` and `__Secure-authjs.session-token`

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
  "@ai-sdk/mcp": "^1.0.25",
  "@ai-sdk/react": "^3.0.118",
  "@civic/mcp-client": "0.1.0",
  "ai": "^6.0.1",
  "jose": "^6.0.11",
  "next": "^16.1.5",
  "next-auth": "5.0.0-beta.30"
}
```

`@ai-sdk/mcp` is a required peer dependency of `@civic/mcp-client` — install it even though you don't import it directly.
