Invoke-RestMethod -Method Put `
  -Uri "http://localhost:9200/_snapshot/local_fs" `
  -ContentType "application/json" `
  -Body '{ "type":"fs", "settings":{ "location":"/snapshots", "compress":true } }'

$ts = Get-Date -Format "yyyyMMdd-HHmm"
Invoke-RestMethod -Method Put `
  -Uri "http://localhost:9200/_snapshot/local_fs/snap-$ts?wait_for_completion=true"

Invoke-RestMethod -Method Get `
  -Uri "http://localhost:9200/_cat/snapshots/local_fs?v" | Out-String

$restore = @{
  indices = "winlogbeat-*,metricbeat-*"
  include_global_state = $false
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
  -Uri "http://localhost:9200/_snapshot/local_fs/snap-$ts/_restore" `
  -ContentType "application/json" `
  -Body $restore
