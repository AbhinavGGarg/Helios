# Helios

**Self-hosted, open-weight autonomous penetration testing. Your code and your findings never leave your machine.**

Helios points a team of AI agents at a web target or a code repository, runs industry-standard security tooling, interprets the raw output with a *local* LLM, reasons about how the findings chain together, and delivers a MITRE ATT&CK–aligned vulnerability report — all streamed live to the UI. Point it at a URL or a repo and it fingerprints the stack, selects only the relevant scanners, validates and de-duplicates findings, builds a causal attack-chain graph, and writes the report.

Unlike cloud pentesting platforms, **nothing is sent to a third-party API.** Helios runs entirely on infrastructure you control — a local Ollama model or your own fine-tuned open-weight model.

---

## Why Helios is different

Autonomous AI pentesting is a crowded space, but every major platform is closed SaaS: you hand your targets, your source code, and your vulnerabilities to someone else's cloud. Helios is the opposite on three axes.

**1. Local & sovereign.** Helios is fully self-hostable and air-gappable. Your source, your targets, and your findings stay on your own machine — the entire pipeline, including LLM inference, runs locally. For security-conscious teams, "we never exfiltrate your codebase" is the whole point.

**2. Trained, not just prompted.** Most tools wrap a general-purpose cloud model. Helios is built to run a *purpose-trained* open-weight model for the task general models are weakest at: turning noisy scanner output into deduplicated, validated, exploit-chained findings. Swap in your own fine-tuned Qwen/Llama checkpoint via the standard provider interface.

**3. Repo-native / shift-left.** Helios scans source repositories directly, not just live web targets — so it can run *before* you ship, as a pre-deployment or PR-time gate, rather than only after something is exposed.

**Explainability as the output.** Helios doesn't just list CVEs. The attack-chain node reasons over the combined findings to produce a causal, MITRE-aligned graph showing how individual weaknesses chain into a realistic kill-chain.

---

## How it works

A **planner node** inspects the target first:

- **URL targets** — a quick fingerprint (httpx + whatweb), then LLM-driven adaptive planning selects the relevant web agents. Under uncertain signal, guardrails keep a safe baseline of recon + SQLi + XSS before the attack-chain and report stages.
- **Repository targets** — walks the file tree, builds a fingerprint, and selects only the relevant agents from `static_c`, `static`, `deps_py`, `deps_js`, `secrets`.

Each selected **agent node** runs its toolset, skips the LLM call if the tools produce no output, and accumulates structured findings into shared graph state. An **attack-chain node** then reasons over the combined findings to produce the MITRE-aligned exploit graph, and a final **report node** synthesises everything into a Markdown report. LLM token streaming is pushed to the frontend over SSE throughout.

## Agent pipeline

| Agent                 | Tools                        | Target type |
| --------------------- | ---------------------------- | ----------- |
| Planner               | LLM + file-tree fingerprint  | both        |
| Recon                 | httpx · nmap · whatweb       | URL         |
| SQL Injection         | sqlmap                       | URL         |
| XSS                   | dalfox                       | URL         |
| C/C++ Static Analysis | cppcheck · semgrep p/c       | repo        |
| Static Analysis       | semgrep · bandit             | repo        |
| Python Deps           | pip-audit                    | repo        |
| JS Deps               | npm audit                    | repo        |
| Secrets               | trufflehog · detect-secrets  | both        |
| Attack Chain          | LLM reasoning (MITRE ATT&CK) | both        |
| Report                | LLM synthesis                | both        |

## LLM providers

Helios defaults to a **local** model so nothing leaves your machine. Set `LLM_PROVIDER` to switch backends:

| Provider           | Env var           | Notes                                  |
| ------------------ | ----------------- | -------------------------------------- |
| `ollama` (default) | `OLLAMA_MODEL`    | Fully local inference                  |
| `openai`           | `OPENAI_MODEL`    | OpenAI-compatible (incl. self-hosted)  |
| `claude`           | `ANTHROPIC_MODEL` | Optional, for comparison               |

To run your own fine-tuned checkpoint, serve it behind any OpenAI-compatible endpoint (e.g. vLLM) and point `OPENAI_BASE_URL` at it.

## Quick start (development)

The backend runs inside Docker (for the security tooling); the frontend runs natively for fast hot-reload. Local inference is served by Ollama on the host.

```bash
# 1. Backend + test target (OWASP Juice Shop)
docker compose -f docker-compose.dev.yml up -d --build

# 2. Frontend
pnpm install
pnpm dev
```

- Frontend: `http://localhost:3000`
- Backend API: `http://localhost:8000`
- Test target: `http://localhost:3001`

For a fully containerised stack (Ollama + backend + frontend + target):

```bash
docker compose up -d --build
```

## Scanning targets

- **Web target:** submit an `http(s)` URL (from inside the backend container, use `http://host.docker.internal:3001` for the bundled Juice Shop).
- **Public repo:** submit a repository root URL; Helios clones it inside the backend container and scans that snapshot — nothing is cloned into your workspace.

## Deploy (live demo)

Helios is built to run locally, but you can stand up a hosted instance for demos. The backend deploys to **Render** and the frontend to **Vercel**.

**Backend (Render).** The repo ships a `render.yaml` blueprint. In Render, create a new Blueprint from this repo; it builds `docker/Dockerfile.backend` as a web service. Set these secrets in the dashboard (they are `sync: false` in the blueprint, never committed): `OPENAI_API_KEY`, `OPENAI_MODEL`, and optionally `CORS_ALLOW_ORIGINS`. A hosted instance has no local Ollama, so it uses an OpenAI-compatible provider (default base URL points at Featherless). Security tooling is memory-heavy, so use at least the `starter` plan.

**Frontend (Vercel).** Deploy the Next.js app and set `BACKEND_API_URL` to your Render backend URL. The built-in `/api/*` proxy forwards browser requests to the backend, so no CORS setup is needed in that path.

> Note: the hosted path trades the "nothing leaves your machine" guarantee for a public demo URL. For real use, self-host with the local Ollama default — that is the point of the project.

## Responsible use

Helios is for testing systems you own or are explicitly authorised to assess. Scanning targets without permission may be illegal. By default, only an allowlist of safe public examples (`example.com`, `scanme.nmap.org`, `testphp.vulnweb.com`) is treated as sanctioned; keep `ENFORCE_TARGET_ALLOWLIST=true` in shared or production deployments.

## License

MIT — see [LICENSE](LICENSE).
