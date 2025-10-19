# Quick end-to-end check (local)
$ErrorActionPreference = 'Stop'
$es  = "http://127.0.0.1:9200"
$kbn = "http://127.0.0.1:5601"

function Wait-HttpOk([string]$url, [int]$timeoutSec=180, [hashtable]$headers=@{}) {
  $deadline = [DateTimeOffset]::UtcNow.AddSeconds($timeoutSec)
  do {
    try {
      $resp = Invoke-WebRequest -Uri $url -TimeoutSec 5 -Headers $headers -SkipHttpErrorCheck
      if ($resp.StatusCode -ge 200 -and $resp.StatusCode -lt 300) { return }
    } catch { Start-Sleep 2 }
  } while ([DateTimeOffset]::UtcNow -lt $deadline)
  throw "Timeout waiting for $url"
}

# 1) Start (falls noch nicht läuft)
docker compose -f compose/docker-compose.yml up -d

# 2) Warten bis ES & Kibana gesund
Wait-HttpOk "$es/_cluster/health?wait_for_status=yellow&timeout=60s" 180
$kbnHeaders = @{ "kbn-xsrf" = "1" }
Wait-HttpOk "$kbn/api/status" 180 $kbnHeaders

Write-Host "✅ ES & Kibana sind erreichbar."

# Optional: wieder stoppen
# docker compose -f compose/docker-compose.yml down
