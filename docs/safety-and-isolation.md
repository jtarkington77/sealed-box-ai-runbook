# Safety, isolation, and remote access

This is the "how to actually lock this thing down" guide.

Assumptions:

- You can install a Linux server or VM.
- You're comfortable in a terminal.
- You can edit config files and restart services.
- You are **not** already a firewall / network expert.

We'll walk through:

- A **baseline sealed-box setup** on a single LAN with an ISP router.
- A **better isolated setup** if you have a real firewall (OPNsense/pfSense/etc.).
- How to reach the box remotely without just port-forwarding your LLM to the internet.
- Outbound controls so the box doesn't quietly spray data out.
- "Safe enough" patterns for data used in RAG and (optional) training.

---

## 1. Threat model

You're building a box that:

- Knows a lot about your life / lab / projects.
- Lives on your network with other important systems.
- Can be driven by code (agents, tools, scripts), not just chat.

We want to make it hard for:

- An external attacker to:
  - Hit an exposed LLM/agent port.
  - Use the box as a pivot into the rest of your network.
- The box to:
  - Leak data out via "helpful" agents or misconfigured tools.
- You to:
  - Accidentally log sensitive data in a way that's easy to steal.

We are not proving anything mathematically secure; we are building something **much safer than "LLM on :8000, port-forwarded to the internet."**

---

## 2. Baseline sealed box (single LAN, ISP router)

This section assumes:

- You have one consumer router from your ISP.
- Everything is on one subnet (for example `192.168.1.0/24`).
- You're running the AI stack on a Linux server called `ai-box`.

### 2.1 Give the AI box a predictable IP

You need the box to have a stable address.

**Option A -- DHCP reservation (preferred if your router supports it)**

1. On `ai-box`, find your MAC address:

        ip a

   Identify the interface that actually has your LAN address (for example `eth0`, `eno1`) and note its MAC.

2. On your router's web UI:
   - Open the DHCP / LAN / address-reservation page.
   - Add an entry:
     - MAC: use the value from step 1.
     - IP: something like `192.168.1.50` (inside your LAN range, outside DHCP pool if required).
   - Save and reboot `ai-box` if needed.

**Option B -- Static IP on the box (example: Ubuntu with netplan)**

Edit `/etc/netplan/01-netcfg.yaml` (file name may differ):

    network:
      version: 2
      renderer: networkd
      ethernets:
        eno1:
          dhcp4: no
          addresses:
            - 192.168.1.50/24
          gateway4: 192.168.1.1
          nameservers:
            addresses:
              - 1.1.1.1
              - 8.8.8.8

Then apply:

    sudo netplan apply

Adjust interface name (`eno1`, `eth0`, etc.) and IPs to match your actual network.

---

## 2.2 Host firewall with UFW (Ubuntu / Debian style)

Goal:

- Default deny incoming.
- Allow only:
  - SSH from LAN (for admin).
  - HTTPS (443) from LAN (for the reverse proxy front door).

Install and configure UFW:

    sudo apt update
    sudo apt install ufw -y

    sudo ufw default deny incoming
    sudo ufw default allow outgoing

Allow SSH and HTTPS from your LAN only (change subnet if needed):

    sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
    sudo ufw allow from 192.168.1.0/24 to any port 443 proto tcp

Enable and verify:

    sudo ufw enable
    sudo ufw status verbose

On other distros you can mirror the same logic with `firewalld` or another host firewall: default deny inbound; allow SSH + 443 from LAN.

---

## 2.3 Container network layout (keep LLM internal)

Goal:

- Only the reverse proxy listens on `0.0.0.0:443`.
- Worker LLM, watchdog, vector DB, etc. sit on a private Docker network.

Example `docker-compose.yml` skeleton:

    version: "3.9"

    networks:
      sealed_ai_net:
        driver: bridge

    services:
      reverse-proxy:
        image: caddy:latest          # or nginx / traefik
        container_name: sealed-box-proxy
        ports:
          - "443:443"                # ONLY public port
        volumes:
          - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
          - ./certs:/certs
        networks:
          - sealed_ai_net
        restart: unless-stopped

      worker-llm:
        image: your-llm-image        # vLLM, Ollama, whatever you choose
        container_name: sealed-box-worker
        expose:
          - "8000"                   # internal only
        networks:
          - sealed_ai_net
        restart: unless-stopped

      watchdog-llm:
        image: your-watchdog-image
        container_name: sealed-box-watchdog
        expose:
          - "8100"                   # internal only
        networks:
          - sealed_ai_net
        restart: unless-stopped

      vector-store:
        image: qdrant/qdrant:latest  # example choice
        container_name: sealed-box-qdrant
        expose:
          - "6333"
        volumes:
          - ./qdrant-data:/qdrant/storage
        networks:
          - sealed_ai_net
        restart: unless-stopped

Notes:

- We use `expose` for worker, watchdog, vector store.
  - That makes them reachable only on the internal bridge (`sealed_ai_net`).
  - There is no `ports:` mapping to the host for these.
- Only `reverse-proxy` has `ports: "443:443"`.

---

## 2.4 Reverse proxy config (Caddy example)

`./caddy/Caddyfile`:

    ai-box.local {
        encode zstd gzip

        # TLS: for LAN you can use internal certs
        tls internal

        # Basic auth (minimum gate; you can swap this to OIDC later)
        basicauth / {
            # user: hashed_password (generate with "caddy hash-password")
            user1 JDJhJDEyJHll...
        }

        @api {
            path /llm* /agent* /rag* /watchdog*
        }

        handle @api {
            reverse_proxy sealed-box-worker:8000
        }

        handle /ui* {
            reverse_proxy some-ui-service:3000
        }

        handle {
            respond "Sealed Box AI front door" 200
        }

        log {
            output file /var/log/caddy/sealed-box-access.log
        }
    }

Key points:

- Caddy listens on 443 and terminates TLS.
- All auth is done here, not by the LLM server directly.
- It only talks to internal containers by name (`sealed-box-worker`, `some-ui-service`, etc.).
- LLM and other services are never bound directly to host ports.

Generate a basic-auth password hash:

    docker run --rm caddy caddy hash-password --plaintext 'your-very-strong-password'

Paste the hash into the `basicauth` block, then restart:

    docker compose up -d reverse-proxy

---

## 2.5 Remote access patterns (no raw ports on the router)

At this point the stack is reachable on:

- `https://ai-box.local` (or `https://192.168.1.50`) from **inside** your LAN only.
- Port 443 on `ai-box` is fronted by Caddy and protected by auth.
- The LLM, watchdog, vector store, etc. are **not** bound to host ports.

You still want to use the web UI (and API) when you're away from home or the office.  
There are three sane ways to do that without just forwarding ports from the router:

1. SSH local tunnel.
2. VPN into your network.
3. HTTPS tunnel (for example Cloudflare Tunnel) that only exposes the reverse proxy.

The common rule:  
**nothing ever exposes the raw LLM service directly -- only the reverse proxy on 443.**

---

### 2.5.1 SSH local tunnel -> web UI

This is the simplest pattern if you're the only user and you're comfortable with SSH.

**What it does**

- Your remote system opens an SSH connection to a host on your network (bastion).
- That SSH connection forwards a local port (for example `8443`) to `ai-box:443`.
- You point a browser at `https://localhost:8443` and you see the web UI exactly as if you were on LAN.

**Steps**

1. Make sure SSH works from the outside to some host you trust.

   This can be:

   - The AI box itself (`ai-box`), or  
   - Another Linux host that can reach `ai-box` on 443.

2. From your remote client system, run something like:

        ssh -L 8443:ai-box.local:443 user@bastion.yourdomain.tld

   Replace:

   - `ai-box.local` with the LAN hostname or IP of the AI box (for example `192.168.1.50`).
   - `user@bastion.yourdomain.tld` with your actual SSH user and host.

3. On that same client system, open your browser and go to:

   - `https://localhost:8443`

   You should see:

   - The Caddy "front door".
   - Then the auth prompt you configured.
   - Then the chat UI / dashboard.

**Why this is safe**

- The only open port on the AI box is 443 on the LAN.
- SSH is the only thing exposed on the WAN (on your bastion).
- The LLM, watchdog, and other services are still only reachable on the internal Docker network.

---

### 2.5.2 VPN into your network

If you want multiple devices to access the box (phone, tablet, other PCs) without constantly running SSH tunnels, a VPN is usually nicer.

**Pattern**

- Run WireGuard / OpenVPN / similar on:
  - Your firewall (OPNsense/pfSense/MikroTik), **or**
  - A small VM/host on a DMZ / separate subnet.
- When you're remote:
  - Connect the VPN from your device.
  - Your device gets an IP on your internal network.
  - You access `https://ai-box.local` (or `https://192.168.1.50`) exactly like you do on-site.

**High-level steps (WireGuard example)**

1. Set up a WireGuard "server" on your firewall or a separate host.
2. Create a peer for each client (phone, workstation, etc.).
3. Configure the VPN to allow access to the LAN where `ai-box` lives.
4. On the client, import the WireGuard config and connect.
5. Browse to the AI box using the same internal URL as at home.

**Why this is safe**

- No ports are forwarded directly from WAN to `ai-box`.
- The AI stack still only listens on the internal LAN.
- You authenticate to the VPN first, then to the AI stack itself.

---

### 2.5.3 HTTPS tunnel with a proxy service (Cloudflare Tunnel example)

If you already use Cloudflare (or a similar service) and don't want to manage a full VPN, you can use an HTTPS tunnel to expose **only** the reverse proxy, with strong auth in front of it.

This pattern assumes:

- You have a domain managed by Cloudflare (for example `yourdomain.com`).
- You can install the Cloudflare tunnel agent (`cloudflared`) on `ai-box` (or a small helper host that can reach it).

> The exact commands change over time; always check Cloudflare's docs.  
> What follows is the **shape** of the setup, not a copy of their manual.

#### a) Install cloudflared

On `ai-box` (or your chosen tunnel host), install the Cloudflare tunnel client using their Linux instructions.  
You should end up with a `cloudflared` binary in your path and a service file under `/etc/cloudflared/` on most distros.

#### b) Log in and create a tunnel

Run:

    cloudflared tunnel login

This opens a browser where you authorize the tunnel client to access your Cloudflare account.

Then:

    cloudflared tunnel create sealed-box-ai

Cloudflare will print a tunnel UUID.  
It also usually drops a credentials file under `/etc/cloudflared/`.

#### c) Configure the tunnel to point at your reverse proxy

Create or edit `/etc/cloudflared/config.yml`:

    tunnel: sealed-box-ai
    credentials-file: /etc/cloudflared/<your-tunnel-uuid>.json

    ingress:
      - hostname: sealedbox.yourdomain.com
        service: https://ai-box.local:443
        originRequest:
          noTLSVerify: true     # because Caddy is using internal TLS
      - service: http_status:404

Replace:

- `sealedbox.yourdomain.com` with the public hostname you want.
- `ai-box.local` with the LAN hostname or IP of your AI box.

This says:

- Cloudflare edge -> your tunnel -> `https://ai-box.local:443` (Caddy).
- Nothing else is exposed; no direct tunnel to the LLM containers.

#### d) Route DNS through the tunnel

From the same host:

    cloudflared tunnel route dns sealed-box-ai sealedbox.yourdomain.com

Cloudflare creates a special DNS record that points `sealedbox.yourdomain.com` to your tunnel.

#### e) Run cloudflared as a service

Enable and start it (exact command depends on how it was installed; typical pattern):

    sudo systemctl enable cloudflared
    sudo systemctl start cloudflared
    sudo systemctl status cloudflared

Once it's running, open a browser outside your network and go to:

- `https://sealedbox.yourdomain.com`

You should see the same Caddy front door and auth flow as on LAN.

#### f) Add another layer of auth (strongly recommended)

Cloudflare offers "Access" / Zero Trust policies that:

- Force SSO login (Google/Microsoft/etc.).
- Limit which accounts can reach `sealedbox.yourdomain.com`.

Use that (or similar features from whatever tunnel provider you use) so that:

- The public URL is not open to the entire internet.
- Even if someone guesses the hostname, they still hit:
  - Tunnel auth (Cloudflare Access), then
  - Your own Caddy auth.

**Why this is safe**

- `ai-box` still only listens on 443 on the LAN.
- No router port forwards.
- Cloudflare sees only HTTPS to Caddy; the LLM service is still internal.
- You get a nice HTTPS URL that works from phones, tablets, and any browser.

---

## 3. Higher isolation (firewall + VLANs)

If you have OPNsense/pfSense/MikroTik/etc., you can go beyond a single flat LAN.

Example layout:

- `LAN` (trusted devices): `192.168.10.0/24`
- `AI_NET` (sealed box segment): `192.168.20.0/24`
- `WAN`: your ISP connection.

`ai-box` sits in `AI_NET` with IP `192.168.20.10`.

### 3.1 Firewall rule patterns

Translate these into your firewall UI.

**From LAN -> AI_NET**

- Allow `LAN` -> `ai-box` on TCP 443 (for HTTPS).
- Optionally allow `LAN` -> `ai-box` on TCP 22 (SSH) from your admin IP.
- Block all other traffic from `LAN` to `AI_NET`.

**From AI_NET -> LAN**

- If needed, allow `ai-box` -> NAS on whatever ports your storage uses (NFS/SMB).  
  If you don't use a NAS, skip this.
- Block all other traffic from `AI_NET` to `LAN`.

**From AI_NET -> WAN**

- Create an alias/group for update hosts (OS repos, container registries, tunnel endpoints).
- Allow `ai-box` -> that alias on TCP 80/443.
- Block all other outbound from `AI_NET` to `WAN`.

**From WAN -> AI_NET**

- No rules that forward WAN traffic directly to `AI_NET`.  
  Remote access must go through:
  - VPN on the firewall or another host, or
  - A tunnel that terminates on the reverse proxy with auth.

---

## 4. App front door: reverse proxy, auth, and API keys

At this point:

- Only 443 is open from clients to the AI box.
- Everything else is on a private container network.

You still need **app-layer** controls.

### 4.1 Single front door

Every part of the stack should sit behind the reverse proxy:

- Chat UI -> `/ui`
- Worker LLM API -> `/llm`
- Agent/orchestration APIs -> `/agent/*`
- Optional watchdog dashboard -> `/watchdog/*`

The LLM and other services should only bind to:

- The Docker bridge network, and/or
- `127.0.0.1` inside the container.

No service should be reachable from the LAN or WAN except via the reverse proxy.

### 4.2 Authentication and API keys

At minimum:

- Require authentication (basic auth, SSO, whatever you choose) for:
  - Chat UI.
  - All `/llm`, `/rag`, `/agent` endpoints.

For programmatic access:

- Issue **per-client API keys**:
  - One key per tool, script, or agent.
  - Store keys in env vars or a secrets manager.
  - Log which key calls which endpoint.

This gives you:

- A way to revoke a specific key if a tool is compromised.
- A way to see which thing generated suspicious traffic.

### 4.3 Admin surfaces

For any admin UI (LLM control panel, vector store console, Grafana, etc.):

- Bind them only on:
  - `127.0.0.1`, or
  - The AI subnet with firewall rules limiting who can reach them.
- Ideally:
  - Reach them via SSH tunnel or VPN only.
  - Do **not** publish them through public tunnels.

---

## 5. Outbound traffic control

Models don't magically phone home, but your code and tools might:

- Agents making HTTP calls.
- Ingestion jobs fetching URLs.
- Scripts sending emails or webhooks.

You want clear answers to:

- "Where can this box talk to--
- "How does data ever leave this subnet--

### 5.1 Network-level outbound

On a real firewall:

- Default deny outbound from `AI_NET`.
- Make small allow rules:
  - OS / package updates.
  - Container registries.
  - Logging / monitoring (if remote).
  - Tunnel endpoints, if you use them.

On an ISP router:

- Use whatever "access control" / outbound rules you have.
- If it's extremely basic:
  - Be conservative with agents that hit the
 internet.
  - Prefer internal-only tools and local data.

### 5.2 Application-level outbound

In your agent/orchestration code:

- Treat "HTTP request" as a **privileged tool**.
- Don't give the model a tool that can hit arbitrary URLs.
- Instead, hard-code allowlists:
  - Your internal docs/wiki.
  - Specific public documentation sites.
  - APIs you control.

For "send email":

- Only allow the agent to send to:
  - You, or a small list of addresses.
- Log:
  - Recipient.
  - Subject.
  - A copy of the body.

This keeps "helpful automation" from quietly turning into "data exfil".

---

## 6. Data for RAG and training

This box is supposed to know your world. That's the whole point.  
You still want to be deliberate about what you feed it.

### 6.1 Safer inputs

Good RAG candidates:

- Your own:
  - Notes and lab writeups.
  - Documentation.
  - Project / study materials.
  - Knowledge base exports.
- Public documentation with:
  - No PII.
  - No sensitive contracts or financials.

Fine-tuning (if you ever do it):

- Use:
  - Generic patterns.
  - Synthetic or scrubbed examples.
  - Data you'd be okay seeing in a controlled demo.

### 6.2 High-risk inputs

Handle these with extreme care, or keep them out entirely:

- Full email archives.
- Client/customer datasets.
- HR, payroll, or medical information.
- Anything that would be a reportable incident if it leaked.

If you insist on using them:

- Keep them on:
  - A separate, encrypted volume.
- Make sure:
  - Only the RAG layer reads them.
  - You don't echo them back into logs.

### 6.3 Logging awareness

Any time you send data into the stack, ask:

- Does this end up in:
  - Web server access logs?
  - LLM server logs?
  - Watchdog logs?
  - Agent logs?
- Are those logs:
  - Rotated?
  - Retained?
  - Backed up somewhere else?

Sometimes the leak is not the model; it's the `logs/` directory you forgot about.

---

## 7. Safety checklists

### 7.1 Minimum "I'm not being reckless"

- [ ] No raw LLM or agent ports exposed on the WAN.
- [ ] Only the reverse proxy listens on 443.
- [ ] Host firewall on `ai-box` is default-deny inbound.
- [ ] Outbound is limited to updates, registries, and a short allowlist.
- [ ] Reverse proxy requires authentication for UI and APIs.
- [ ] Watchdog model sees request/response pairs and logs its verdicts.
- [ ] Logs live somewhere you can actually review them.

### 7.2 "I care about this a lot"

Everything above, plus:

- [ ] AI stack lives on its own subnet/VLAN.
- [ ] Firewall rules strictly control:
  - LAN -> AI_NET
  - AI_NET -> LAN
  - AI_NET -> WAN
- [ ] Remote access only via VPN / SSH tunnel / secure tunnel endpoint on the proxy.
- [ ] Agents use allowlisted tools only; no arbitrary HTTP by default.
- [ ] RAG sources are intentional and documented.
- [ ] Logs are rotated, backed up, and included in your incident-response plan.

---

From here, plug into:

- `docs/watchdog-monitoring.md` for wiring the smaller model into the logging path.
- `docs/agents-and-tools.md` for concrete examples of how worker + watchdog + tools actually play together.




