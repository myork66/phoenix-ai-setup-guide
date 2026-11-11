# A Practical Guide to AI Chat: Cloud-Hosted vs. Private Local Hosting

**Public, sanitized edition — 11 November 2026**  
**Scope:** conversational AI with an optional Open WebUI front end. The local architecture is modeled on a multi-container, security-conscious Linux installation; it is **not** an export of anyone's real deployment.

**License:** Written documentation: [CC BY 4.0](LICENSE.md). Original code and configuration examples: [MIT](LICENSE-CODE.md). Third-party projects retain their own licenses.

> **Privacy note:** All paths, labels, loopback ports, and example values here are generic. No real API key, hostname, credential, public IP, VPN relay, model directory, private firewall address, or user's deployment fingerprint is included. Generate your own secrets. Do not publish `docker inspect`, `.env`, secrets, debug logs, or browser API-token screenshots unredacted.

## Before choosing

| Topic | 1. Cloud-hosted model | 2. Locally hosted model |
|---|---|---|
| Model computation | Provider's infrastructure | Your hardware |
| Network needed to chat | Usually yes | No, after software and model are obtained |
| Setup effort | Low | Moderate to advanced |
| Hardware | Browser-capable device | Linux host with adequate RAM/VRAM |
| Usage cost | Subscription and/or API billing | Electricity, hardware and maintenance |
| Privacy boundary | Prompts reach provider | Prompts can remain on host **if all connected services also remain local** |
| Maintenance | Provider maintains inference stack | Operator maintains OS, Docker, model and app |
| Interface | Provider website/app, or optional Open WebUI | Open WebUI through localhost |

**Do not confuse:** Open WebUI is an interface, not an inference engine. Installing it on your PC does **not** make an external model local. The inference endpoint determines where the text is processed.

---

# 1. Conventional cloud-hosted AI — step by step

## Path 1A — simplest: provider's website or application

1. **Select a provider.** Compare its model offerings, contractual terms, regional availability, privacy/data-use policy, expected costs, and features. The list of providers and plans changes; consult their current official documentation.
2. **Create an account** on the provider's official website or app. Enable multi-factor authentication if offered.
3. **Choose the appropriate plan.** A consumer chat subscription is generally **not** the same as API credit or API entitlement; verify separately if an API is needed.
4. **Review privacy controls.** Learn whether chats are retained, used for training, exportable, or deletable; these policies depend on product, plan, and settings.
5. **Start a new chat** and select an available model.
6. **Test with a low-risk prompt:** `Explain the difference between a private LAN and the public internet in three sentences.`
7. **Adjust output behavior** with ordinary prompts (e.g., `Use three sentences`, `Show assumptions`, `Give a concise checklist`).
8. **Enable only needed capabilities** such as browsing, document processing, or connected apps. These may route data to additional services.
9. **Maintain account security:** unique password, MFA, review connected devices and active sessions.
10. **Verify costs and usage** regularly if using a metered plan.

**Verification:** the app responds; the active account/model and its data-control settings are known; billing limits match your expectations.

## Path 1B — cloud inference, but with the same Open WebUI interface

This is optional. It provides a self-operated *interface* while model inference remains **cloud-hosted**. It requires Docker; use the official Docker installation documentation for your operating system.

1. Install Docker Engine and the Compose plugin, or Docker Desktop where appropriate. Verify `docker compose version`.
2. In a new folder, create `compose.yaml`:

```yaml
services:
  webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "127.0.0.1:3000:8080"
    environment:
      WEBUI_AUTH: "true"
      ENABLE_OPENAI_API: "true"
    volumes:
      - webui_data:/app/backend/data
    restart: unless-stopped

volumes:
  webui_data:
```

3. Start it with `sudo docker compose up -d`.
4. Visit `http://127.0.0.1:3000` and create the **first** user; this user becomes administrator. Confirm new signups are disabled afterward.
5. Obtain a **separately billed API key** from your chosen model provider, if that provider requires one. Do not paste it into public terminals, screenshots, or logs.
6. Open **Settings → Admin → Connections → Manage OpenAI API Connections**. Add an external connection. For an OpenAI API example, the official endpoint is `https://api.openai.com/v1`; for another provider, use its **official** compatible endpoint. Enter the key privately in the UI.
7. Save/verify the connection and select a model from the chat model selector. If the provider does not support model-list detection, add its documented model ID manually.
8. Test a short prompt, inspect the provider's usage/billing, and confirm the selected model is the intended cloud model.
9. For stopping/starting the interface, use `sudo docker compose stop` and `sudo docker compose start` in that folder.

**Security limitation:** this is a convenient localhost-bound UI, not an air-gapped system. Requests, submitted context and potentially metadata travel to the external provider. The Open WebUI database also resides on the local disk. For a sustained deployment, pin a tested image version/digest, use a stable WebUI secret and back up the volume.

---

# 2. Locally hosted AI — security-conscious Linux example

## Architecture

```text
Browser on this PC
    |
    | http://127.0.0.1:18083 (host loopback only)
    v
Open WebUI  [Docker app network]
    |
    | OpenAI-compatible request with API key
    v
NGINX proxy [app network + isolated backend network]
    |
    | authorization passed to llama.cpp
    v
llama.cpp server [isolated backend network only]
    |
    v
GGUF model file [read-only mount]
```

The proxy can also be exposed **only to localhost** at `127.0.0.1:18082` for authenticated API tests. This example does **not** publish llama.cpp's own port on the host.

This reproducible example demonstrates the major layers of a Phoenix-style design. It is deliberately **not claimed to recreate** an audited, encrypted-home, multi-firewall, VPN-gated production system without further host-specific engineering and verification.

## Prerequisites

- Linux system with a functioning GPU driver (the example assumes a Vulkan-capable GPU). For NVIDIA CUDA, CPU-only, Intel SYCL, or other backends, change to the supported inference image/device configuration.
- Sufficient RAM, storage, and VRAM for the selected licensed GGUF model, its quantization and context length. Model hardware demands vary substantially; do not assume a 27B model fits every GPU.
- Working Docker Engine with `docker compose` plugin; `curl`, `openssl`, and a text editor.
- A legally obtained, trusted GGUF file compatible with llama.cpp; verify source, model license and published checksum when available.
- An account with `sudo` permission. Avoid permanent Docker group membership unless its root-level privilege implications are understood.

## Step 1 — Install and verify Docker

For Ubuntu-family Linux distributions, follow the current **official** installation guide rather than copying repository instructions from an unknown blog:

- https://docs.docker.com/engine/install/ubuntu/
- https://docs.docker.com/compose/install/linux/

Then verify:

```bash
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

**Checkpoint:** Docker daemon and Compose plugin work. For distributions other than Ubuntu, use the appropriate official Docker instructions; do not blindly reuse the Ubuntu repository commands.

## Step 2 — Check GPU render-device availability

```bash
ls -l /dev/dri/
```

For the Vulkan example below, verify that `/dev/dri/renderD128` exists and corresponds to your intended GPU. On multi-GPU systems it may be a different render node; inspect rather than guess. A host Vulkan check such as `vulkaninfo --summary` may help (install the distribution's Vulkan tools if needed).

**Checkpoint:** the intended GPU works at the host level, and the correct render node is known. If not, fix driver/device permissions *before* starting the containers.

## Step 3 — Create private directories

```bash
umask 077
mkdir -p "$HOME/local-ai/models" \
         "$HOME/local-ai/secrets" \
         "$HOME/local-ai/data/open-webui" \
         "$HOME/local-ai/nginx"
cd "$HOME/local-ai"
```

These are **generic example paths** that can be changed consistently. Restrict access to the configuration and secret folders.

## Step 4 — Install a licensed model

Download a compatible GGUF file from a trusted source, review its license, then place it at:

```text
~/local-ai/models/model.gguf
```

The name `model.gguf` is illustrative: rename the actual file *or* change the Compose path to match. Never place provider keys inside the model directory.

**Checkpoint:** `ls -lh ~/local-ai/models/model.gguf` shows the expected file, size and location. Validate its published SHA-256 hash if supplied by its distributor.

## Step 5 — Create two independent secrets

```bash
cd "$HOME/local-ai"
umask 077
openssl rand -hex 32 > secrets/llama_api_key
openssl rand -hex 32 > secrets/webui_secret_key
chmod 600 secrets/llama_api_key secrets/webui_secret_key
```

The API key authorizes access to llama.cpp. The WebUI secret signs sessions and protects certain stored credentials. They are **different secrets**. Never paste either into public issue reports. Preserve both securely across restarts and upgrades.

## Step 6 — Create a minimal proxy configuration

Create `~/local-ai/nginx/default.conf` with:

```nginx
server {
    listen 8080;
    server_name _;

    location / {
        resolver 127.0.0.11 valid=10s ipv6=off;
        set $llama_backend "llama";
        proxy_pass http://$llama_backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Authorization $http_authorization;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 3600s;
        proxy_buffering off;
    }
}
```

Make the configuration *readable by the unprivileged NGINX container*:

```bash
chmod 644 "$HOME/local-ai/nginx/default.conf"
```

This proxy forwards the `Authorization` header to llama.cpp; it does **not** independently authenticate clients. The llama.cpp server must be started with its `--api-key-file` option as shown next. Do not connect it to a public-facing port.

## Step 7 — Create `compose.yaml`

```yaml
name: local-ai-example

services:
  llama:
    image: ghcr.io/ggml-org/llama.cpp:server-vulkan
    command:
      - --model
      - /models/model.gguf
      - --ctx-size
      - "4096"
      - --n-gpu-layers
      - "20"
      - --parallel
      - "1"
      - --host
      - 0.0.0.0
      - --port
      - "8080"
      - --api-key-file
      - /run/secrets/llama_api_key
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    volumes:
      - ./models:/models:ro
      - ./secrets/llama_api_key:/run/secrets/llama_api_key:ro
    networks:
      - backend
    restart: "no"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  proxy:
    image: nginx/nginx-unprivileged:stable-alpine-slim
    ports:
      - "127.0.0.1:18082:8080"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - backend
      - app
    depends_on:
      - llama
    restart: "no"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  webui:
    image: ghcr.io/open-webui/open-webui:main-slim
    ports:
      - "127.0.0.1:18083:8080"
    environment:
      WEBUI_AUTH: "true"
      WEBUI_SECRET_KEY_FILE: /run/secrets/webui_secret_key
      ENABLE_OLLAMA_API: "false"
      ENABLE_OPENAI_API: "true"
      OPENAI_API_BASE_URL: http://proxy:8080/v1
      OFFLINE_MODE: "true"
      HF_HUB_OFFLINE: "1"
      ENABLE_VERSION_UPDATE_CHECK: "false"
      WEBUI_URL: http://127.0.0.1:18083
      CORS_ALLOW_ORIGIN: http://127.0.0.1:18083;http://localhost:18083
    volumes:
      - ./data/open-webui:/app/backend/data
      - ./secrets/webui_secret_key:/run/secrets/webui_secret_key:ro
    networks:
      - app
    depends_on:
      - proxy
    restart: "no"

networks:
  backend:
    internal: true
  app:
    driver: bridge
```

**Read before running:**

- Replace the model filename/path and render-node path if your system differs.
- `--n-gpu-layers 20` is an **initial example**, not a benchmarked optimum. Adjust after checking memory and logs; set `0` for CPU-only only with an appropriately configured CPU server image and without the GPU-device mapping.
- `main-slim`, `server-vulkan` and `stable-alpine-slim` are moving tags used for an approachable first run. After validation, **pin tested image digests** in a maintained production deployment.
- The Docker `backend` network is isolated from normal external routing, but `app` is a normal Docker bridge and may have outbound connectivity. `OFFLINE_MODE` limits application behavior, **not** network egress. See hardening section.
- Do not disable WebUI signup *before* first-user creation unless you have deliberately configured an alternative administrator bootstrap method.
- For Linux hosts with SELinux, custom user namespaces, rootless Docker, or restrictive filesystem ownership, container-to-file access needs additional distribution-specific handling.

## Step 8 — Validate YAML without starting anything

```bash
cd "$HOME/local-ai"
sudo docker compose config --quiet
sudo docker compose config --services
```

Expected service names: `llama`, `proxy`, `webui`. A valid YAML file does not guarantee the GPU or the model will work; these are separate tests.

## Step 9 — Start model and proxy, then check logs

```bash
sudo docker compose up -d llama proxy
sudo docker compose ps
sudo docker compose logs --tail=80 llama proxy
```

Wait until llama.cpp reports that the model is loaded and serving. If GPU initialization or out-of-memory fails, check the render device, host driver, offload-layer count and model size before proceeding.

## Step 10 — Verify API authentication

A call **without** a valid API key should be rejected by the model API (often HTTP `401`; do not insist on one code without checking your server version):

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  http://127.0.0.1:18082/v1/models
```

To test with the key **without displaying it in the command line or terminal history**:

```bash
(
  set -euo pipefail
  printf 'header = "Authorization: Bearer %s"\n' \
    "$(<"$HOME/local-ai/secrets/llama_api_key")" |
  curl --config - --silent --show-error --fail-with-body \
    http://127.0.0.1:18082/v1/models
)
```

Avoid sharing the full response if model identifiers or other deployment metadata are sensitive. If the unauthenticated call succeeds unexpectedly, **stop and repair authentication before continuing**.

## Step 11 — Start Open WebUI and create the administrator

```bash
cd "$HOME/local-ai"
sudo docker compose up -d webui
sudo docker compose ps
```

Open `http://127.0.0.1:18083` on that host. Create the first administrator, use a strong password, and verify that subsequent signup is disabled under **Settings → Admin → Authentication**. The first startup may take longer with a fresh UI/database.

## Step 12 — Attach the local model connection

1. Open **Settings → Admin → Connections**.
2. Under **Manage OpenAI API Connections**, add or edit the connection.
3. Use **Internal** / OpenAI-compatible connection if the UI offers that choice.
4. URL: `http://proxy:8080/v1` (**inside** Docker, not `localhost`).
5. Authentication: **Bearer**; use the locally generated `llama_api_key` through the private admin UI.
6. Save and verify the connection, then choose the returned model from the model selector. If necessary, add the model ID shown by `/v1/models`.
7. Send a concise test: `In exactly two sentences, distinguish a firewall from a VPN.`

**Checkpoint:** response arrives through Open WebUI, the model appears in the selector, and llama.cpp logs show inference without authentication failures.

## Step 13 — Verify local-only published endpoints

```bash
sudo docker compose ps
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Expected host-published endpoints:

- Open WebUI: `127.0.0.1:18083` → container port `8080`
- API proxy: `127.0.0.1:18082` → container port `8080`
- llama.cpp: **no host-published port**

Test from a **different device** on the LAN that these services are not accessible at the host's LAN address. Avoid routing, reverse proxies, VPN port forwards, or tunnel services that expose localhost to other machines without deliberate access controls.

## Step 14 — Daily operations

From `~/local-ai`:

| Action | Command |
|---|---|
| Start existing containers | `sudo docker compose start` |
| Initial deployment or deliberate re-create | `sudo docker compose up -d` |
| Stop all without deleting | `sudo docker compose stop` |
| Show status | `sudo docker compose ps` |
| Inspect recent logs | `sudo docker compose logs --tail=80` |
| Validate config | `sudo docker compose config --quiet` |

Keep the directory and secret files backed up securely. Avoid `docker compose down -v`, `docker volume prune`, and indiscriminate `docker system prune` as routine shutdown or cleanup commands.

## Step 15 — Hardening from a working baseline

This step is **not optional if you need a high-assurance privacy claim**, but the exact implementation is host-specific. Add each control separately and test after each change:

1. **Pin image digests** only after validating specific versions; document image, version and rollback target.
2. **Reduce privileges:** dedicated non-root UID/GID when compatible; GPU render group supplemental permissions; read-only root filesystems and tmpfs mounts where tested; drop capabilities; `no-new-privileges`.
3. **Protect Docker administration:** do not expose the Docker socket or daemon; avoid adding general users to the powerful `docker` group.
4. **Enforce egress restrictions:** apply host-level `DOCKER-USER`/nftables rules or another *verified* network boundary to block unwanted app-network and backend egress. Docker loopback-published ports protect inbound access; they do **not** block outbound traffic.
5. **Test fail-closed behavior:** while services are running, deliberately remove the allowed external route in a maintenance window and verify connections fail as designed. Test IPv4 and IPv6 where enabled, and validate across reboots and Docker restarts.
6. **Control startup order:** if models and credentials live in an encrypted home, do not start containers before the directory is mounted; use a properly tested systemd unit with preflight checks, or manual start after login. `restart: "no"` only disables *Docker's automatic restart policy*; it does not implement a preflight by itself.
7. **Local data protection:** full-disk or home encryption protects files at rest when mounted data is unavailable; it is not a barrier against a compromised logged-in session.
8. **WebUI access:** disable unauthorized signups, restrict tools/code execution/web access unless needed, use a strong admin password, and limit any remote access behind an authenticated HTTPS/VPN layer.
9. **Backups and recovery:** include Compose files, model versions/checksums, WebUI database, keys, proxy config and firewall policy; store encrypted backups apart from the active machine. Never restore an old database against an unreviewed new version.
10. **Observe performance:** record cache, prompt token count, prompt processing time, output tokens and output time. A short prompt with thousands of tokens may indicate excessive injected tool definitions or historical context; a lengthy answer may simply use more generation time. Audit the actual request rather than assuming.

### Optional AI profiles

- **Fast:** little/no reasoning if supported by the specific model and server, no unneeded built-in tools, concise prompts.
- **Analytical:** deliberate reasoning if the backend supports it; tools enabled selectively; compare answer quality against additional time.

Reasoning controls are **model- and server-specific**. A parameter such as `reasoning_effort=none` is not universally supported; do not blindly copy it across models. Always test the actual output and timings.

## Step 16 — Troubleshooting and rollback

| Symptom | First checks |
|---|---|
| Model missing from UI | API URL must be `http://proxy:8080/v1` inside the container; inspect `/v1/models` and bearer credentials |
| Proxy error | `docker compose logs proxy llama`; NGINX upstream resolution, isolated network membership |
| Authentication rejected | Confirm correct local API key file and connection auth; do **not** print secrets into tickets |
| GPU not used | Render node, host driver, Vulkan runtime, device access, `--n-gpu-layers` |
| OOM or startup failure | Smaller GGUF/quantization, fewer offloaded layers, smaller context |
| UI loses settings | Confirm persistent data mount and unchanged WebUI signing secret |
| Very long first answer | Inspect prompt context size, cache reuse, tool definitions and output length |
| UI responds but remote LAN device reaches it | Check actual Docker published bind, host forwarding, VPN tunnels and reverse proxies |

**Safe basic rollback:** `sudo docker compose stop`, restore only the previously backed-up configuration/image digest, then start and validate. Database migrations can make downgraded app versions incompatible; retain tested database backups.

---

# Relation to other Docker services

A broader self-hosted stack may also contain a grammar service, metasearch, privacy proxy, monitoring dashboard, or media automation. **These are independent products**, not prerequisites for the two AI setup paths above.

If using download clients or a VPN kill-switch, treat them as a separate project with their own network topology, firewall policy, and start/stop procedures. Connecting a VPN on the host does **not** prove that every Docker container is tunneled or leak-proof.

For managing multiple containers, a graphical Docker console can show status, but its individual **Start** button does not necessarily execute systemd preflights, Compose dependency ordering, or host firewall tests. Use the project's documented control command for coordinated systems.

---

# Public-release safety checklist

- [ ] No API keys, passwords, tokens or private-key material
- [ ] No personal home paths, hostnames, real external IPs, VPN relay details or unique network identifiers
- [ ] No actual `docker inspect` environment blocks or secrets from screenshots
- [ ] Example images/tags are reviewed and pinned before production use
- [ ] Model license and redistribution terms have been checked
- [ ] The text does not claim `OFFLINE_MODE` prevents outbound network traffic
- [ ] Cloud usage and third-party data handling are clearly identified
- [ ] Local setup is tested on the target OS/GPU instead of assumed portable

## Official documentation and references (verified November 2026)

- Docker Engine for Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Compose plugin: https://docs.docker.com/compose/install/linux/
- Docker privilege model: https://docs.docker.com/engine/install/linux-postinstall/
- llama.cpp official container images: https://github.com/ggml-org/llama.cpp/blob/master/docs/docker.md
- llama.cpp server flags and auth: https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
- Open WebUI installation: https://docs.openwebui.com/getting-started/quick-start/
- Open WebUI connecting models: https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/
- Open WebUI environment variables: https://docs.openwebui.com/reference/env-configuration/
- Open WebUI security hardening: https://docs.openwebui.com/getting-started/advanced-topics/hardening/
- Open WebUI offline-mode limitations: https://docs.openwebui.com/tutorials/maintenance/offline-mode/
- NGINX unprivileged image: https://github.com/nginx/docker-nginx-unprivileged

**Confidence:** High in the architectural distinctions and documented configuration interfaces; moderate in cross-hardware portability. The local Compose example has been reviewed for consistency, **not end-to-end executed on a newly provisioned host**. Treat it as an implementation template and validate each checkpoint.

---

# License and reuse

This project is intentionally **dual-licensed by material type**:

| Material | License |
|---|---|
| Original guide prose, tables, and explanatory diagrams | [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE.md) |
| Original shell commands, scripts, Docker Compose YAML, NGINX config, and other code/configuration examples | [MIT License](LICENSE-CODE.md) |
| Third-party images, tools, dependencies, model weights, documentation, and trademarks | Their respective upstream licenses and terms |

**Suggested documentation attribution:** “Phoenix AI Setup Guide” by [@myork66](https://github.com/myork66), licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Source: https://github.com/myork66/phoenix-ai-setup-guide. Indicate if you made changes.

For code/configuration reuse, preserve the MIT copyright and license notice as required by [LICENSE-CODE.md](LICENSE-CODE.md). License grants extend only to material contributors have authority to license; publication does not relicense third-party software or model files.
