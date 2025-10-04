Invoke-RestMethod -Method Put `
  -Uri "http://localhost:9200/_snapshot/local_fs" `
  -ContentType "application/json" `
  -Body '{"type":"fs","settings":{"location":"/usr/share/elasticsearch/snapshots","compress":true}}'

$ts = Get-Date -Format "yyyyMMdd-HHmm"
Invoke-RestMethod -Method Put -Uri "http://localhost:9200/_snapshot/local_fs/snap-$ts?wait_for_completion=true"

Invoke-RestMethod -Method Get -Uri "http://localhost:9200/_cat/snapshots/local_fs?v"
