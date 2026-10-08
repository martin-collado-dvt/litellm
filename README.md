# Native microservices UI prefix browser comparison

PR: https://github.com/BerriAI/litellm/pull/45316

Before source: `7a659973e362673d130e33f51b2a4332644b8260` (PR merge-base).
After source: `6f82d81bd5c44e8f55db6de0f0ae6a439fe5eefe` (PR tip).

Both UI images were built from the unmodified `ui/Dockerfile` at the named source revisions on a native amd64 GitHub runner. The before build is https://github.com/martin-collado-dvt/litellm/actions/runs/37764253115; the after build is https://github.com/martin-collado-dvt/litellm/actions/runs/37758739527.

The browser URL for both captures is `http://localhost:43046/services/llm/ui/`. Only the UI container image changes. Backend, gateway, database, cache, ingress configuration and authenticated browser session remain the same. The ingress publishes no origin-root auxiliary routes. `ingress.conf` is the shared test reverse proxy configuration, not a patch to either UI image.

The real backend, gateway and migration images are official `v1.104.1` components. PostgreSQL is `16.11-alpine`; Valkey is `9.1.2`. The dataset contains no models or provider credentials. No provider calls are made. Credentials are generated locally and excluded from these artifacts.

Both UI containers run with `SERVER_ROOT_PATH=/services/llm`, uid/gid `10001:10001`, a read-only root filesystem and writable `/tmp`.

1. Start PostgreSQL and Valkey; run the official migration component successfully.
2. Start backend and gateway with the same `SERVER_ROOT_PATH`, database and cache configuration. Use `model_list: []` and `STORE_MODEL_IN_DB=True`.
3. Start the after UI and ingress; sign in interactively with locally generated test credentials.
4. Replace only the UI with the before image. Reload the test ingress to resolve the replacement container, then open the comparison URL. The static page lacks its CSS/JavaScript and does not reach the dashboard; accessibility output stays on Loading.
5. Replace only the UI with the after image. Reload the test ingress, wait for `/healthz`, and open the exact same URL. The existing session reaches the admin dashboard.
6. Navigate to Models + Endpoints and Virtual Keys and reload directly. Navigation, assets and backend configuration stay below `/services/llm`.

`before.jpg` and `after.jpg` are original browser screenshots. The corresponding SVG files embed those exact bytes and add only red boxes around the affected UI. No page pixels are retouched.
