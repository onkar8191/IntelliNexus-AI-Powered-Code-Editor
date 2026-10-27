# NexaCode

NexaCode is a browser coding workspace and interview practice platform, built in stages from the supplied product brief. The current repository contains a React/Vite frontend, a TypeScript API, PostgreSQL persistence, an in-browser SQLite workbench, optional Judge0 execution, a cited RAG tutor, and stdio plus Streamable HTTP MCP servers.

## Local setup

Requirements: Node.js 20+, npm, and Docker Desktop with Compose. Run `npm test` for core unit checks and `npm run build` for a production build.

1. Install dependencies: `npm install`
2. Create local config: `Copy-Item .env.example .env`
3. Start the app database: `docker compose up -d postgres`
4. Start both apps: `npm run dev`
5. Open the Vite URL printed in the terminal (usually `http://localhost:5173`). The Express API runs at `http://localhost:4000`; Vite proxies `/api` requests to it.

Postgres starts from `db/init.sql` with the user, problem, submission, notes, bookmark, study-plan, and pgvector knowledge tables. The API seeds its current starter problems when it connects. If you already have a Postgres volume from an earlier schema, apply the SQL additions in `db/init.sql` manually or recreate only your local dev volume.

## Optional services

### Sandboxed multi-language execution

The JavaScript editor can run code in a short-lived Web Worker. Other selected languages use the API, which only forwards code to Judge0; it never executes user code in Express.

1. Copy `judge0.conf.example` to `judge0.conf` and replace all placeholder passwords/tokens.
2. Put the same Judge0 authentication token in `.env` as `JUDGE0_AUTH_TOKEN`, and set `JUDGE0_URL=http://localhost:2358`.
3. Start the Judge0 profile: `docker compose --profile execution up -d judge0-db judge0-redis judge0-server judge0-worker`.

Judge0 requires privileged containers for its isolation runtime. Keep its API bound to localhost behind the NexaCode API. The included configuration disables user code network access and compiler/command-line overrides. Judge0's official Compose deployment uses separate server, worker, PostgreSQL, and Redis services; this repository follows that layout ([official Judge0 Compose file](https://github.com/judge0/judge0/blob/master/docker-compose.yml)).

### AI Tutor and knowledge ingestion

The AI tutor requires a configured OpenAI-compatible chat/embedding provider, or local Ollama. Add provider settings to `.env`; `AI_API_KEY` stays on the server. The vector store has 1536 dimensions; smaller local embedding vectors are zero-padded, preserving cosine similarity.

```sh
npm run ingest
```

Ingestion reads Markdown in `data/knowledge`, splits it with overlap, skips unchanged files, and stores embeddings in pgvector. The tutor endpoint is authenticated, rate-limited, streams SSE events, includes source labels, treats retrieved material and code as untrusted, redacts common secret formats, and declines when retrieved context is weak.

### MCP

`npm run mcp` starts the local stdio server for an MCP client. Build first if using the generated executable: `npm run build:api`. `npm run mcp:http` starts the Streamable HTTP endpoint on localhost port 4001. Set `MCP_AUTH_TOKEN` before binding it to a network interface and restrict `MCP_ALLOWED_ORIGINS`. The VS Code connection example is in `.vscode/mcp.json`; see [docs/MCP.md](docs/MCP.md) for tools and transport setup. MCP clients should request user consent before calling `run_code`.

## Implemented now

- Monaco editor, language modes for 12 languages, local drafts, font controls, responsive workspace and light/dark styling.
- 150 original practice problems, topic and difficulty search, progressive hints, local notes/bookmarks, and progress view.
- JavaScript execution in a two-second Worker; other languages forward to Judge0 when configured.
- SQLite WASM in a Web Worker with sample employee/order/library tables, query history, result grid, read-only filtering, and CSV/JSON export.
- Express API with Zod validation, security headers, CORS allowlist, request/rate limits, bcrypt password hashing, httpOnly JWT cookies, email/password and GitHub OAuth sign-in, and persistent notes/bookmarks/submissions/progress endpoints.
- Judge0 adapter with CPU/memory bounds and verdict mapping.
- RAG ingestion/retrieval, streaming tutor route with citations, and tutor panel.
- MCP tools for problem search/details, Judge0 runs, read-only SQL, user progress, recommendations, and knowledge search; problem/schema/study-plan resources and three prompt templates.
- Authenticated tutor-side MCP calls for problem and knowledge lookup, with explicit Allow/Deny confirmation before code execution; remote tool calls are rate-limited and recorded with redacted input summaries.
- CI workflow for typecheck and production builds.

## Not complete yet

This is not yet the entire product scope in the brief. All 150 interview problems have JavaScript/TypeScript function-based visible and hidden tests plus tested JavaScript reference solutions. Python, Java, C, C++, C#, Go, Rust, PHP, Ruby, and Kotlin problem runs/submissions use a standalone-program contract: each test case is provided as one JSON object on stdin, and the program must print its JSON result to stdout. The editor provides language-specific starter scaffolds; reference solutions are currently JavaScript only. GitHub OAuth is implemented by stable GitHub account ID; configure its three `GITHUB_*` values and register the exact callback URL. The SQL workbench includes 50 prompt/reference-query exercises against a small synthetic dataset. The account study-plan schedule, weekly leaderboard, streaming cited tutor, and retrieval hit-rate evaluation are implemented. Tutor MCP tools are currently user-triggered rather than selected autonomously by an LLM agent; broader automated tests and cross-language function wrappers remain. The sample SQL datasets are intentionally small. Local browser storage is retained as a fallback; account sync is not wired into every UI element yet.

No user code is run in the Express process. The in-browser JavaScript Worker is useful for a demo, but production isolation for arbitrary code depends on a correctly secured Judge0 deployment.



### GitHub sign-in

Create a GitHub OAuth App with callback URL `http://localhost:4000/api/auth/github/callback`, then set `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, and the exact `GITHUB_CALLBACK_URL` in `.env`. Production must use its HTTPS callback URL. GitHub identities are stored by GitHub account ID; an email match alone never links accounts.
