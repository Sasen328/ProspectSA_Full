# ProspectSA — Free Production Deploy (Oracle + Supabase + Cloudflare)

**Total cost: $0/mo forever** (optionally $9/yr for a `.com` domain).
**Total time: ~3 hours of clicking + copy-paste.**

## Architecture

```
┌──────────────────────────┐    ┌────────────────────────────┐    ┌──────────────────┐
│ Cloudflare Pages         │───▶│ Oracle Cloud VM (Free)     │───▶│ Supabase         │
│ React frontend           │    │ Docker: api + scout        │    │ Postgres + Auth  │
│ <your>.pages.dev         │    │ api.<your-domain>          │    │ (managed, free)  │
└──────────────────────────┘    └────────────────────────────┘    └──────────────────┘
       FRONTEND                        API + SCRAPERS                    DATABASE
```

| Service              | Tier         | Limit                                        |
|----------------------|--------------|----------------------------------------------|
| Oracle Cloud Ampere  | Always Free  | 4 ARM cores, 24GB RAM, 200GB disk, forever   |
| Supabase             | Free         | 500MB Postgres, 1GB storage, 50K MAU         |
| Cloudflare Pages     | Free         | Unlimited bandwidth, 500 builds/mo           |
| Cloudflare DNS       | Free         | Unlimited                                    |
| Let's Encrypt (Caddy)| Free         | Auto-renewing HTTPS                          |

---

## PHASE 0 — Prerequisites (do these first, in parallel)

### 0.1 Sign up for Oracle Cloud Always Free
1. Go to https://signup.cloud.oracle.com
2. Use your real name + a working email + a credit card (it WILL NOT be charged on Always Free — they hold for verification)
3. **Home Region:** pick the region nearest you. **This cannot be changed later.** For Saudi Arabia, use **Saudi Arabia Central (Riyadh)** or **UAE East (Dubai)**.
4. Wait for the verification email (5 min to 24 hours). Account becomes usable when you see the OCI Console.

### 0.2 Sign up for Supabase
1. https://supabase.com → "Start your project"
2. Sign in with GitHub
3. Create new project:
   - Name: `prospectsa`
   - Database Password: **save this somewhere safe**
   - Region: closest to your Oracle region
4. Wait ~2 min for provisioning.

### 0.3 Sign up for Cloudflare
1. https://dash.cloudflare.com/sign-up
2. Free plan is enough — no credit card needed for Pages.

### 0.4 (Optional) Buy a domain
- $9/yr at https://www.cloudflare.com/products/registrar/
- Or skip and use the free `<project>.pages.dev` subdomain.

---

## PHASE 1 — Set up the Oracle VM

### 1.1 Create the instance
1. OCI Console → **Compute → Instances → Create Instance**
2. Name: `prospectsa-vm`
3. **Image:** Canonical Ubuntu 22.04 (default)
4. **Shape:** click "Change shape" → **Ampere (ARM)** → **VM.Standard.A1.Flex**
   - OCPUs: **4**
   - Memory: **24 GB**
5. **Networking:** "Create new VCN" → leave defaults, but check **"Assign a public IPv4 address"**
6. **SSH keys:** click "Generate a key pair for me" → **download both files** (keep `.key` safe — it's your private key).
7. **Boot volume:** size to **100 GB** (free tier allows 200GB total).
8. Click **Create**. Wait ~2 min for `Running` state. Note the **public IP**.

### 1.2 Open ports 80 + 443 in the VCN
1. OCI Console → **Networking → Virtual Cloud Networks → your VCN → Security Lists → Default**
2. Add ingress rules:
   - Source CIDR `0.0.0.0/0`, TCP, dest port `80`
   - Source CIDR `0.0.0.0/0`, TCP, dest port `443`

### 1.3 SSH into the VM
```bash
# from your Codespace terminal (where you downloaded the .key file)
chmod 600 ssh-key-*.key
ssh -i ssh-key-*.key ubuntu@<PUBLIC_IP>
```

### 1.4 Install Docker + tools (on the VM)
```bash
# As ubuntu user on the VM:
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg git
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker ubuntu
# log out and back in for group to apply
exit
```

### 1.5 Open OS firewall ports
```bash
# back on the VM after re-ssh
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save  # or: sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

---

## PHASE 2 — Set up Supabase as the database

### 2.1 Get the connection string
1. Supabase dashboard → your project → **Settings → Database**
2. Scroll to **Connection string** → select **URI** format → **Transaction** mode (port 6543)
3. Copy. It looks like:
   ```
   postgresql://postgres.xxxxxxxxxxxx:[YOUR-PASSWORD]@aws-0-us-east-1.pooler.supabase.com:6543/postgres
   ```
4. Replace `[YOUR-PASSWORD]` with the password you set during project creation.

### 2.2 Apply our schema to Supabase
On the VM (after cloning the repo, see Phase 3.1):
```bash
# Install psql client
sudo apt-get install -y postgresql-client

# Set DATABASE_URL to your Supabase URI
export DATABASE_URL='postgresql://postgres.xxx:YOUR_PASS@aws-0-...pooler.supabase.com:6543/postgres'

# Apply migrations (in order)
psql "$DATABASE_URL" -f lib/db/drizzle/0000_*.sql
psql "$DATABASE_URL" -f lib/db/drizzle/0001_*.sql
psql "$DATABASE_URL" -f lib/db/drizzle/0002_missing_tables.sql

# Load seed data
psql "$DATABASE_URL" -f seed_data.sql
```

### 2.3 Disable Supabase RLS for now
- Supabase enables Row-Level Security by default. We don't use it yet.
- Dashboard → **Authentication → Policies** → for each table click "Disable RLS" (or just for `companies`, `executives`, `leads`, `lead_factory_jobs`, `lead_lists`).

---

## PHASE 3 — Deploy the API + Scout to Oracle

### 3.1 Clone the repo on the VM
```bash
# As ubuntu on the VM
git clone https://github.com/Sasen328/ProspectSA_Full.git
cd ProspectSA_Full
git checkout main  # or whichever branch is your production line
```

### 3.2 Create production .env
```bash
nano .env
```

Paste this template (fill in your real values):

```bash
# --- Database (Supabase) ---
DATABASE_URL=postgresql://postgres.xxx:YOUR_PASS@aws-0-...pooler.supabase.com:6543/postgres
PGSSLMODE=require

# --- App ---
NODE_ENV=production
PORT=3000
FRONTEND_ORIGIN=https://<your-project>.pages.dev

# --- LLM keys (paste your rotated keys) ---
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
PERPLEXITY_API_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
OPENROUTER_API_KEY=
TAVILY_API_KEY=

# --- Nexus: prefer free OpenRouter models ---
NEXUS_PREFER_FREE_MODELS=true

# --- Optional ---
HUGGINGFACE_API_KEY=
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=  # from Settings → API
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

### 3.3 Drop Postgres service from compose (using Supabase now)
On the VM:
```bash
# Make a production override that skips the local postgres service
cat > docker-compose.prod.yml <<'EOF'
services:
  postgres:
    # Disabled in prod — using Supabase
    profiles: ["never"]
  app:
    restart: unless-stopped
    depends_on: []
    ports:
      - "127.0.0.1:3000:3000"
EOF
```

### 3.4 Build + start
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml build app
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d app
docker compose logs -f app
```

Wait for "Server listening on :3000". `Ctrl+C` to stop tailing.

Verify:
```bash
curl http://127.0.0.1:3000/api/health
```

### 3.5 Add Caddy for HTTPS + reverse proxy
```bash
# Install Caddy
sudo apt-get install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt-get update && sudo apt-get install -y caddy

# Configure
sudo nano /etc/caddy/Caddyfile
```

Paste (replace `api.your-domain.com`):
```
api.your-domain.com {
    reverse_proxy 127.0.0.1:3000 {
        flush_interval -1   # don't buffer SSE
    }
}
```

```bash
sudo systemctl reload caddy
```

---

## PHASE 4 — Deploy the frontend to Cloudflare Pages

### 4.1 Connect repo
1. Cloudflare dashboard → **Workers & Pages → Create application → Pages → Connect to Git**
2. Authorize GitHub → pick `Sasen328/ProspectSA_Full`
3. Project name: `prospectsa`
4. Production branch: `main`

### 4.2 Build settings
- **Framework preset:** None
- **Build command:**
  ```
  corepack enable && pnpm install --frozen-lockfile && pnpm --filter @workspace/prospect-sa build
  ```
- **Build output directory:** `artifacts/prospect-sa/dist`
- **Root directory:** leave blank (repo root)

### 4.3 Environment variables (Cloudflare Pages → Settings → Environment variables)
- `VITE_API_BASE` = `https://api.your-domain.com`
- `NODE_VERSION` = `20`

### 4.4 Deploy
Click **Save and Deploy**. Wait ~3-5 min for first build.

You'll get a URL like `https://prospectsa.pages.dev`. Test it.

---

## PHASE 5 — DNS + domain wiring (only if you bought a domain)

### 5.1 Point domain at Cloudflare nameservers
- If you bought via Cloudflare Registrar: already done.
- If elsewhere: change nameservers to the two Cloudflare gives you.

### 5.2 DNS records
- Cloudflare → DNS → Records → Add record:
  - **A record:** `api` → `<Oracle VM public IP>` → **Proxy: OFF** (orange cloud GRAY — Caddy handles TLS directly)
- For the frontend custom domain: Pages → Custom domains → add `app.your-domain.com` (Cloudflare auto-configures).

### 5.3 Update CORS + frontend
- On the VM, edit `.env` → set `FRONTEND_ORIGIN=https://app.your-domain.com`
- `docker compose restart app`
- On Cloudflare Pages → env vars → update `VITE_API_BASE=https://api.your-domain.com` → redeploy.

---

## PHASE 6 — Verify the full stack

```bash
# From any browser:
curl https://api.your-domain.com/api/health           # → {"ok":true}
curl https://api.your-domain.com/api/companies?limit=5 # → JSON of 5 SA companies

# Open https://app.your-domain.com → log in → run a lead factory job → watch SSE stream.
```

---

## ONGOING — Update flow

```bash
# On the VM
cd ~/ProspectSA_Full
git pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml build app
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d app

# Cloudflare Pages auto-deploys on every push to main.
```

---

## What we DROPPED to fit free tier

| Dropped                          | Why                              | Replacement                                     |
|----------------------------------|----------------------------------|-------------------------------------------------|
| Local Postgres in compose        | Move to Supabase                 | Supabase managed Postgres (free 500MB)          |
| Heavy Playwright Scout (optional)| Saves ~1GB image                 | Keep `ENABLE_SCOUT=true` env to re-enable later |
| Render / Railway / Fly           | Not free / paid only             | Oracle Always Free ARM VM                       |

## What stays the same

- Full 7-agent Lead Factory pipeline ✅
- Nexus LLM router with free OpenRouter models ✅
- All harvesters (Tavily, Google News, Saudi RSS, sanctions) ✅
- All Lead Factory exports (CSV/XLSX/PDF/PPT/JSON) ✅
- Signal Intel + Relationship Intel ✅
- AI Chat ✅
- ProsEngine, OrcEngine, MasarEngine ✅

---

## When something breaks

| Symptom                                  | Fix                                                              |
|------------------------------------------|------------------------------------------------------------------|
| `curl api.../health` times out           | Check Oracle VCN security list ports 80/443 open                 |
| `502 Bad Gateway` from Caddy             | `docker compose logs app` — API didn't start                     |
| CORS error in browser console            | `.env` `FRONTEND_ORIGIN` must match Pages URL exactly            |
| DB connect refused                       | Supabase URI: use port **6543** (pooler), not 5432               |
| Build OOM on Oracle                      | `NODE_OPTIONS="--max-old-space-size=4096"` before docker build   |
| Caddy can't get cert                     | DNS not propagated yet — wait 5-10 min or use Cloudflare DNS API |

---

## Next session — code changes I'll do once your Oracle VM is up

1. **Slim Dockerfile**: multi-stage, ARM-compatible build target (`linux/arm64`)
2. **Make Scout optional**: split into its own compose service behind `profiles: [scout]`
3. **Supabase migration runner**: single `pnpm run migrate:supabase` script
4. **Production docker-compose.prod.yml** committed to repo
5. **Caddyfile** committed to repo
6. **GitHub Actions** to auto-build the ARM image on push (optional)

Ping me when your Oracle VM is created and you have the public IP — I'll do the code changes then.
