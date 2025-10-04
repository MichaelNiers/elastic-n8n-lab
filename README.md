# elastic-n8n-lab

## Setup
```powershell
wsl -d docker-desktop sysctl -w vm.max_map_count=262144
cd compose
copy .env.example .env
docker compose up -d
