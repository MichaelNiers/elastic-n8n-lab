# elastic-n8n-lab

Reproducible lab with **Elasticsearch** + **Kibana** (Docker Compose), plus helper scripts/notes.

---

## Contents
- [Repository layout (quick view)](#repository-layout-quick-view)
- [Quick start](#quick-start)
  - [Windows (Docker Desktop + WSL2)](#windows-docker-desktop--wsl2)
  - [Linux (bare metal / VPS)](#linux-bare-metal--vps)
  - [Update / Upgrade](#update--upgrade)
- [Status & logs](#status--logs)
- [Compose validation (syntax/render)](#compose-validation-syntaxrender)
- [Health checks](#health-checks)
- [Snapshots (Elasticsearch)](#snapshots-elasticsearch)
- [Kibana: import saved objects](#kibana-import-saved-objects)
- [.env — key variables](#env--key-variables)
- [Notes](#notes)

---

## Repository layout (quick view)

```text
elastic-n8n-lab/
├─ compose/
│  ├─ caddy/
│  │  └─ Caddyfile.example
│  ├─ docker-compose.yml
│  ├─ .env.example          # template (copy to .env)
│  └─ .env                  # local only; in .gitignore
├─ kibana/
│  └─ saved_objects.ndjson  # for Kibana import
├─ docs/
│  ├─ BACKUP_RESTORE.md
│  ├─ MIGRATION.md
│  └─ RUNBOOKS.md
├─ logs/                    # local start/runtime logs (git-ignored)
├─ snapshots/               # ES snapshots (git-ignored, has .gitkeep)
├─ .gitignore
└─ README.md
```

### Quick start

> Run all `docker compose` commands inside `elastic-n8n-lab/compose`.

### Windows (Docker Desktop + WSL2)

#### 0) Clone the repo (skip if already cloned)

```powershell
git clone https://github.com/MichaelNiers/elastic-n8n-lab.git
cd elastic-n8n-lab\compose
```
#### 1) One-time: raise vm.max_map_count inside the Docker VM (WSL2 backend required)

```powershell
wsl -d docker-desktop sysctl -w vm.max_map_count=262144

wsl -d docker-desktop cat /proc/sys/vm/max_map_count
```

#### 2) Configure & start

```powershell
copy .env.example .env
docker compose pull
docker compose up -d
```

### Linux (bare metal / VPS)

#### 0) Clone the repo

```powershell
git clone https://github.com/MichaelNiers/elastic-n8n-lab.git
cd elastic-n8n-lab/compose
```

#### 1) One-time: persistent vm.max_map_count on the host

```powershell
"vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-elastic.conf | Out-Null
sudo sysctl -w vm.max_map_count=262144
```

#### Verify

```powershell
sysctl vm.max_map_count
```

#### 2) Configure & start

```powershell
cp .env.example .env
docker compose pull
docker compose up -d
```

#### Update / Upgrade

1. Edit `.env` and set `ELASTIC_VERSION=x.y.z`.
2. Pull images and restart.

```powershell
cd "$HOME/elastic/elastic-n8n-lab/compose"

docker compose pull
docker compose up -d
docker compose ps
```

### Status & logs

#### Show container status

```powershell
docker compose ps
```

#### Follow logs

```powershell
docker compose logs -f es01
docker compose logs -f kib01
```

#### Stop the stack

```powershell
docker compose down
```

#### Optional: stop & remove volumes for a clean reset

```powershell
docker compose down -v
```

### Compose validation (syntax/render)

```powershell
cd $HOME\elastic\elastic-n8n-lab\compose
docker compose config > $env:TEMP\compose.rendered.yml
"Rendered Compose: $env:TEMP\compose.rendered.yml"
```

### Health checks 

```powershell
$ES = "http://localhost:9200"
$KB = "http://localhost:5601"
```

#### Verify ES version

```powershell
(Invoke-RestMethod "$ES").version.number
```

```powershell
# If you're on Windows PowerShell 5.1, add -UseBasicParsing to Invoke-WebRequest
# $r = Invoke-WebRequest "$KB/api/status" -UseBasicParsing -Headers @{ "kbn-xsrf"="1" } -TimeoutSec 5

try {
  $i = Invoke-RestMethod "$ES" -TimeoutSec 5
  "ES reachable → version=$($i.version.number) cluster=$($i.cluster_name)"
} catch { "ES NOT reachable → $($_.Exception.Message)" }

try {
  $r = Invoke-WebRequest "$KB/api/status" -Headers @{ "kbn-xsrf"="1" } -TimeoutSec 5
  "Kibana HTTP status: $($r.StatusCode)"
} catch { "Kibana NOT reachable → $($_.Exception.Message)" }
```

### Snapshots (Elasticsearch)

#### Register filesystem snapshot repo (one-time)

```powershell
Invoke-RestMethod -Method Put `
  -Uri "http://localhost:9200/_snapshot/local_fs" `
  -ContentType "application/json" `
  -Body '{ "type":"fs", "settings":{ "location":"/snapshots", "compress":true } }'
```

#### Create a snapshot (timestamped)

```powershell
$ts = Get-Date -Format "yyyyMMdd-HHmm"
Invoke-RestMethod -Method Put -Uri "http://localhost:9200/_snapshot/local_fs/snap-$ts?wait_for_completion=true"
```

#### List snapshots

```powershell
Invoke-RestMethod -Method Get -Uri "http://localhost:9200/_cat/snapshots/local_fs?v" | Out-String
```

#### Restore a snapshot

##### A) Restore a specific snapshot (set ID manually)

```powershell
$snapId = "<SNAPSHOT_ID>"

$restore = @{
  indices               = "winlogbeat-*,metricbeat-*"
  include_global_state  = $false
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
  -Uri "http://localhost:9200/_snapshot/local_fs/$snapId/_restore" `
  -ContentType "application/json" -Body $restore
```

##### B) Restore the most recent snapshot automaticall

```powershell
$last = (Invoke-RestMethod -Uri "http://localhost:9200/_cat/snapshots/local_fs?format=json" |
         Sort-Object -Property start_epoch -Descending |
         Select-Object -First 1).id

$restore = @{
  indices               = "winlogbeat-*,metricbeat-*"
  include_global_state  = $false
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
  -Uri "http://localhost:9200/_snapshot/local_fs/$last/_restore" `
  -ContentType "application/json" -Body $restore
```

##### C) Restore and rename

```powershell
$last = (Invoke-RestMethod "http://localhost:9200/_cat/snapshots/local_fs?format=json" |
         Sort-Object -Property start_epoch -Descending |
         Select-Object -First 1).id

$restore = @{
  indices               = "winlogbeat-*,metricbeat-*"
  include_global_state  = $false
  rename_pattern        = "(.+)"
  rename_replacement    = "restored-$1"
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
  -Uri "http://localhost:9200/_snapshot/local_fs/$last/_restore" `
  -ContentType "application/json" -Body $restore
```

### Kibana: import saved objects

1. Open Kibana → Stack Management → Saved Objects → Import

2. Select file: kibana/saved_objects.ndjson

### .env — key variables

```text
ELASTIC_VERSION=
ELASTIC_SECURITY_ENABLED=   # dev: false
ELASTIC_SECURITY_SSL=       # dev: false
ES_JAVA_OPTS=               # e.g., -Xms512m -Xmx512m
KIBANA_MEM_MB=              # e.g., 512
```

### Security (optional)

#### 1) In `.env`:
```text
ELASTIC_SECURITY_ENABLED=true
ELASTIC_SECURITY_SSL=false     # keep SSL off for local first run
ELASTIC_PASSWORD=<strong-password>
# Uncomment only if your compose expects them:
# KIBANA_ES_USER=kibana_system
# KIBANA_ES_PASS=<strong-password>
```

#### Restart the stack:

```powershell
cd $HOME\elastic\elastic-n8n-lab\compose
docker compose up -d
```