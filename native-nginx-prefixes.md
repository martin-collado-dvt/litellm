# Native nginx overlap reproduction

Source: PR revision `6f82d81bd5c44e8f55db6de0f0ae6a439fe5eefe`

The unchanged startup preparation and nginx configuration were tested against the official native nginx 1.31.5 image. Both suggested overlapping prefixes work. No operator configuration patch was applied

Run from a checkout of that source revision:

```sh
docker run --rm --name litellm-prefix-repro \
  --user 10001:10001 --read-only --tmpfs /tmp:rw,size=32m \
  -p 127.0.0.1:43047:3000 -e SERVER_ROOT_PATH=/ui \
  -v "$PWD/ui:/ui:ro" \
  nginx:1.31.5-alpine3.24@sha256:34f40471dea485273c5e2a04dd5e97a682332ceb4a9adecd67de450dcb2fb390 \
  sh -ec '
    mkdir -p /tmp/source/login /tmp/source/_next
    printf "<html><head></head><body>login</body></html>" > /tmp/source/login/index.html
    printf "<html><head></head><body>dashboard</body></html>" > /tmp/source/index.html
    printf "prefix-asset\n" > /tmp/source/_next/app.js
    sh /ui/prepare-root-path.sh /tmp/source /tmp/runtime /ui/nginx.conf /tmp/runtime/nginx.conf "$SERVER_ROOT_PATH"
    exec nginx -c /tmp/runtime/nginx.conf -g "daemon off;"
  '
```

After startup, from another terminal:

```text
curl -s -w '\nHTTP %{http_code}\n' http://localhost:43047/ui/ui/login/
<html><head><meta name="litellm-server-root-path" content="/ui"></head><body>login</body></html>
HTTP 200

curl -s -w '\nHTTP %{http_code}\n' http://localhost:43047/ui/ui/
<html><head><meta name="litellm-server-root-path" content="/ui"></head><body>dashboard</body></html>
HTTP 200
```

Stop the reproduction container and run the same command with `SERVER_ROOT_PATH=/_next`. After startup:

```text
curl -s -w '\nHTTP %{http_code}\n' http://localhost:43047/_next/_next/app.js
prefix-asset

HTTP 200

curl -s -w '\nHTTP %{http_code}\n' http://localhost:43047/_next/ui/login/
<html><head><meta name="litellm-server-root-path" content="/_next"></head><body>login</body></html>
HTTP 200
```

These are real nginx requests using a minimal static export fixture. They isolate the rewrite claim rather than replacing the full-stack browser comparison
