---
title: "Serving Ollama properly: nginx, TLS, and remote access"
tags: [ollama, nginx, tls, systemd, reverse-proxy, self-hosting]
series_part: 2
toc: true
description: "Move Ollama from a localhost experiment to a properly served API: nginx reverse proxy, TLS termination, authentication, and remote access with a hardened systemd unit."
---

This is part 2 of the [Ollama infra series](/series/#ollama-infra-series). [Part 1](/posts/ollama-infra-series/from-zero/) covered installation and the basic API. Here we harden the setup for use by other machines on your network or the internet: nginx in front, TLS, a shared secret for access control, and a systemd unit that survives reboots and crashes.

---

## Why put nginx in front

Ollama's built-in server is not designed to be internet-facing. It has no TLS support, no authentication, and no rate limiting. nginx handles all of that and more:

- TLS termination (your clients talk HTTPS, nginx talks HTTP to Ollama on localhost)
- A shared secret header as a lightweight auth layer
- Request logging in a format you can feed into observability tools later
- Connection limits and timeouts
- A stable public address that does not change if you move Ollama to a different port

---

## Binding Ollama to localhost only

By default on Linux, after the install script runs, Ollama binds to `0.0.0.0:11434` — any interface. Change it to listen only on localhost so nothing reaches it directly:

```bash
systemctl edit ollama
```

This opens an override file. Add:

```ini
[Service]
Environment="OLLAMA_HOST=127.0.0.1:11434"
```

Save and reload:

```bash
systemctl daemon-reload
systemctl restart ollama
```

Verify:

```bash
ss -tlnp | grep 11434
# should show 127.0.0.1:11434, not 0.0.0.0:11434
```

From this point, nothing reaches Ollama directly except nginx running on the same machine.

---

## nginx configuration

Install nginx if you do not have it:

```bash
apt install nginx
```

Create `/etc/nginx/sites-available/inference`:

```nginx
upstream ollama_backend {
    server 127.0.0.1:11434;
    keepalive 8;
}

server {
    listen 80;
    server_name inference.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name inference.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/inference.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/inference.yourdomain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Long timeouts — LLM inference takes time
    proxy_read_timeout    300s;
    proxy_connect_timeout 10s;
    proxy_send_timeout    300s;

    # Required for streaming responses
    proxy_buffering       off;
    proxy_cache           off;

    location /direct/ {
        if ($http_x_ollama_key != "your-secret-here") {
            return 403;
        }
        rewrite ^/direct/(.*) /$1 break;
        proxy_pass         http://ollama_backend;
        proxy_http_version 1.1;
        proxy_set_header   Connection "";
        proxy_set_header   Host localhost;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/inference-access.log json_combined;
    error_log  /var/log/nginx/inference-error.log;
}
```

Enable and test:

```bash
ln -s /etc/nginx/sites-available/inference /etc/nginx/sites-enabled/inference
nginx -t
systemctl reload nginx
```

---

## TLS with Let's Encrypt

If your server has a public domain pointing at it:

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d inference.yourdomain.com
```

Certbot will modify your nginx config to add the certificate paths and set up auto-renewal via a systemd timer.

For a private network (no public domain), use a self-signed certificate or a wildcard cert from your internal CA. With self-signed certs, clients must either trust the CA or disable certificate verification — only do the latter on private networks.

---

## Hardening the systemd unit

The default Ollama unit works but has no restart policy or resource limits. A complete override:

```bash
systemctl edit ollama
```

```ini
[Service]
Environment="OLLAMA_HOST=127.0.0.1:11434"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_NUM_PARALLEL=4"
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
```

Relevant environment variables:

| Variable | What it controls |
|---|---|
| `OLLAMA_HOST` | Bind address |
| `OLLAMA_MAX_LOADED_MODELS` | How many models stay in memory simultaneously |
| `OLLAMA_NUM_PARALLEL` | Concurrent requests per model |
| `OLLAMA_KEEP_ALIVE` | How long a model stays loaded after last request (default: 5m) |
| `OLLAMA_MAX_QUEUE` | Max queued requests before 503 |

`OLLAMA_KEEP_ALIVE` controls the trade-off between cold-start latency and memory use. For a server that handles bursts, set it to `30m` or longer. For a memory-constrained machine with multiple models, set it to `5m` or `0`.

---

## Calling the remote API

From any machine with network access, add the secret header:

```bash
$ curl https://inference.yourdomain.com/direct/api/generate -s -H "X-Ollama-Key: your-secret-here"   -d '{
    "model": "llama3.2:3b",
    "prompt": "What is a keepalive?",
    "stream": false
  }' | jq
{
  "model": "llama3.2:3b",
  "created_at": "2026-04-21T09:54:18.622159665Z",
  "response": "A keepalive, in computing and networking contexts, refers to a periodic message or signal sent ...",
  "done": true,
  "done_reason": "stop",
  "context": [...],
  "total_duration": 26965557054,
  "load_duration": 255380905,
  "prompt_eval_count": 31,
  "prompt_eval_duration": 450871626,
  "eval_count": 357,
  "eval_duration": 25763962391
}
```

With the Python client:

```python
import httpx

client = httpx.Client(
    base_url="https://inference.yourdomain.com",
    headers={"X-Ollama-Key": "your-secret-here"},
    timeout=120,
)
response = client.post("/direct/api/generate", json={
    "model": "llama3.2:3b",
    "prompt": "What is a keepalive?",
    "stream": False,
})
print(response.json()["response"])
```

---

## Log format for observability

nginx's default combined log format is fine, but a JSON format is easier to parse later when we add structured logging in [part 7](/posts/ollama-infra-series/observability/). Add to `/etc/nginx/nginx.conf` inside the `http` block:

```nginx
log_format json_combined escape=json
    '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"x_forwarded_for":"$http_x_forwarded_for",'
    '"request_id":"$request_id",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$uri",'
    '"args":"$args",'
    '"protocol":"$server_protocol",'
    '"status":$status,'
    '"bytes_sent":$bytes_sent,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"request_time":"$request_time",'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_response_time":"$upstream_response_time",'
    '"upstream_connect_time":"$upstream_connect_time",'
    '"upstream_header_time":"$upstream_header_time",'
    '"referer":"$http_referer",'
    '"user_agent":"$http_user_agent"'
    '}';
```

Then in your inference server block:

```nginx
access_log /var/log/nginx/inference-access.log json_combined;
```

Each request now lands as a JSON object — easy to tail, easy to ship to a log aggregator, easy to query with `jq`.

An example is shown below:

```bash
$ tail -f /var/log/nginx/inference-access.log
{
  "time": "2026-04-21T12:04:25+02:00",
  "remote_addr": "127.0.0.1",
  "x_forwarded_for": "",
  "request_id": "32a558cfdd53f4e2134e48e572d8e306",
  "host": "inference.yourdomain.com",
  "method": "POST",
  "uri": "/api/generate",
  "args": "",
  "protocol": "HTTP/1.1",
  "status": 200,
  "bytes_sent": 4091,
  "body_bytes_sent": 3905,
  "request_time": "25.246",
  "upstream_addr": "127.0.0.1:11434",
  "upstream_status": "200",
  "upstream_response_time": "25.246",
  "upstream_connect_time": "0.001",
  "upstream_header_time": "25.246",
  "referer": "",
  "user_agent": "curl/8.5.0"
}
```

---

## What you have now

- Ollama bound to localhost, unreachable directly
- nginx in front with TLS and a shared secret
- A hardened systemd unit that restarts on failure
- JSON access logs ready for the observability post
- Remote clients can call the API from anywhere with the right credentials
