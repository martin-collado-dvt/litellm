# Standalone UI custom path reproduction

This branch contains reproduction artifacts only, not a proposed application change

The official `ghcr.io/berriai/litellm-ui:v1.104.1` image is served under `/services/llm`. The reverse proxy strips that prefix and denies other paths, reproducing a prefix-scoped HTTPRoute without requiring Kubernetes, a database, credentials or an LLM provider

```sh
docker compose up -d
curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:8080/services/llm/ui/login
```

Open `http://localhost:8080/services/llm/ui/login`. The HTML loads but requests CSS, JavaScript and fonts under `/litellm-asset-prefix/_next/` at the origin root. Those requests return 404 and the page stays on Loading

The Docker Compose recipe is provided for reviewers but has not been executed on the reporting machine because no Docker daemon is available. The attached evidence was obtained with untouched HTML/JS/CSS extracted from the official image and its original nginx configuration, using native nginx 1.28.0. Only filesystem paths and the listening port were adjusted for the local runtime. A second nginx server applies the prefix strip and returns 404 for requests outside the prefix. No backend, SSO or provider was mocked or exercised

Local browser URL: `http://127.0.0.1:43020/services/llm/ui/login`

```text
GET /services/llm/ui/login                                                   HTTP 200
GET /litellm-asset-prefix/_next/static/chunks/0ckv5j_fni3xm.css                 HTTP 404
GET /services/llm/litellm-asset-prefix/_next/static/chunks/0ckv5j_fni3xm.css    HTTP 200
```

The hashed CSS filename above is evidence from this image, not a stable API. `browser-result.json` records the browser requests and response statuses

![Official UI served under a scoped prefix](subpath-loading.png)

Cleanup for the Docker recipe:

```sh
docker compose down
```
