# homelab

homelab
├── apps
│   ├── local-path-provisioner
│   │   ├── application.yaml
│   │   └── kustomization.yaml
│   │
│   ├── pihole
│   │   ├── application.yaml
│   │   ├── kustomization.yaml
│   │   └── values.yaml
│   │
│   ├── influxdb
│   │   ├── app
│   │   │   ├── deployment.yaml
│   │   │   ├── kustomization.yaml
│   │   │   ├── pvc.yaml
│   │   │   └── service.yaml
│   │   ├── application.yaml
│   │   └── kustomization.yaml
│   │
│   └── grafana
│       ├── application.yaml
│       ├── kustomization.yaml
│       └── values.yaml
│
├── argocd-apps
│   └── kustomization.yaml
│
├── bootstrap
│   └── root-application.yaml
│
└── README.md

## Pihole Admin-Passwort

Das Pihole-Admin-Passwort wird **nicht** im Klartext in `apps/pihole/values.yaml`
abgelegt, sondern aus einem bestehenden Kubernetes-Secret gelesen
(`admin.existingSecret`). Das Secret liegt nur im Cluster, nicht im Git.

Secret einmalig anlegen (Namespace `pihole` muss existieren – ArgoCD legt ihn per
`CreateNamespace=true` an):

```bash
# Namespace ggf. vorab anlegen, falls das Secret vor dem ersten Sync gesetzt wird
kubectl create namespace pihole --dry-run=client -o yaml | kubectl apply -f -

# Passwort setzen (ersetze DEIN_SICHERES_PASSWORT)
kubectl create secret generic pihole-admin-password \
  --namespace pihole \
  --from-literal=password='DEIN_SICHERES_PASSWORT' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Nach dem Anlegen/Ändern des Secrets den Pihole-Pod neu starten, damit das neue
Passwort übernommen wird:

```bash
kubectl rollout restart deployment/pihole -n pihole
```


## Grafana Admin-Zugang

Analog zu Pihole wird das Grafana-Admin-Passwort **nicht** im Klartext in
`apps/grafana/values.yaml` abgelegt, sondern aus einem bestehenden
Kubernetes-Secret gelesen (`admin.existingSecret`). Das Secret liegt nur im
Cluster, nicht im Git.

Secret einmalig anlegen (Namespace `grafana` muss existieren – ArgoCD legt ihn
per `CreateNamespace=true` an):

```bash
# Namespace ggf. vorab anlegen, falls das Secret vor dem ersten Sync gesetzt wird
kubectl create namespace grafana --dry-run=client -o yaml | kubectl apply -f -

# Admin-User und Passwort setzen (ersetze DEIN_SICHERES_PASSWORT)
kubectl create secret generic grafana-admin-credentials \
  --namespace grafana \
  --from-literal=admin-user='admin' \
  --from-literal=admin-password='DEIN_SICHERES_PASSWORT' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Grafana wird über einen MetalLB-`LoadBalancer`-Service erreichbar gemacht. Die
zugewiesene IP findest du mit:

```bash
kubectl get svc -n grafana grafana
```

> Hinweis: Das OSS-Grafana-Helm-Chart wurde von `grafana.github.io/helm-charts`
> nach `grafana-community.github.io/helm-charts` migriert. Die Application nutzt
> bereits das neue Repo.

## Proxmox-Monitoring (InfluxDB + Grafana)

Ablauf der Metriken:

```
Proxmox-Host ──push──> InfluxDB (k8s, MetalLB-IP) <──query── Grafana
```

Proxmox pusht seine Metriken nativ über das eingebaute "Metric Server"-Feature
an InfluxDB. Es ist kein Exporter nötig. Grafana liest dieselben Daten
cluster-intern aus InfluxDB und zeigt sie im Proxmox-Dashboard an.

### 1. InfluxDB-LoadBalancer-IP herausfinden

ArgoCD rollt InfluxDB automatisch aus (Datenbank `proxmox` wird beim Start
angelegt). Die IP, an die Proxmox pushen soll:

```bash
kubectl get svc -n influxdb influxdb
```

Die `EXTERNAL-IP` ist die MetalLB-Adresse (im Folgenden `<INFLUXDB_IP>`).

### 2. Proxmox Metric Server konfigurieren

In der Proxmox-Weboberfläche: **Datacenter → Metric Server → Add → InfluxDB**

- **Name:** z. B. `homelab-influx`
- **Server:** `<INFLUXDB_IP>`
- **Port:** `8086`
- **Protocol:** `HTTP`
- **Organization/Bucket bzw. Database:** `proxmox`
- InfluxDB-Version/Typ: **InfluxDB (v1 / HTTP)** – passend zum InfluxDB-1.11-Image

Alternativ per CLI auf dem Proxmox-Host:

```bash
pvesh create /cluster/metrics/server/homelab-influx \
  --type influxdb \
  --server <INFLUXDB_IP> \
  --port 8086 \
  --influxdbproto http \
  --organization proxmox \
  --bucket proxmox
```

Nach wenigen Sekunden beginnt Proxmox zu pushen. Prüfen (vom Proxmox-Host oder
einem Rechner mit Zugriff):

```bash
curl -G "http://<INFLUXDB_IP>:8086/query?db=proxmox" \
  --data-urlencode "q=SHOW MEASUREMENTS"
```

Erscheinen Measurements wie `system`, `cpustat`, `memory`, fliessen die Daten.

### 3. Grafana-Dashboard

Grafana provisioniert automatisch:

- die Datenquelle **InfluxDB-Proxmox** (`apps/grafana/values.yaml`)
- das Dashboard **"Proxmox via InfluxDB"** (grafana.com ID `15356`)

Nach dem Login (siehe Grafana Admin-Zugang) liegt das Dashboard direkt bereit.

> Hinweis zu Auth: InfluxDB laeuft hier ohne HTTP-Auth und ist nur im LAN per
> MetalLB-IP erreichbar. Willst du Auth, setze `INFLUXDB_HTTP_AUTH_ENABLED=true`
> im InfluxDB-Deployment, lege User/Passwort an und ergaenze die Credentials
> sowohl in der Grafana-Datenquelle als auch im Proxmox Metric Server.

## HTTPS / Ingress (*.homelab.lan)

Die Web-Dienste sind ueber HTTPS unter `*.homelab.lan` erreichbar. Umgesetzt mit
ingress-nginx (MetalLB-IP `192.168.1.246`) und cert-manager mit einer eigenen,
selbstsignierten Homelab-CA.

| Dienst        | URL                          |
|---------------|------------------------------|
| Actual Budget | https://actual.homelab.lan   |
| Grafana       | https://grafana.homelab.lan  |
| Pihole Admin  | https://pihole.homelab.lan/admin |

Wichtige Details:

- **DNS:** Die Hostnamen werden von Pihole aufgeloest (`customDnsEntries` in
  `apps/pihole/values.yaml` -> `192.168.1.246`). Clients muessen Pihole als
  DNS-Server nutzen, damit `*.homelab.lan` aufgeloest wird.
- **Actual Budget / SharedArrayBuffer:** Der Ingress setzt die Header
  `Cross-Origin-Opener-Policy` und `Cross-Origin-Embedder-Policy`. Damit ist die
  Seite "cross-origin isolated" und `SharedArrayBuffer` funktioniert. Das geht
  nur ueber HTTPS – direkter HTTP-Zugriff per IP fuehrt zum Fehler.
- **Zertifikat / Browser-Warnung:** Die CA ist selbstsigniert. Fuer ein gruenes
  Schloss die CA einmalig in Browser/OS importieren. Export der CA:
  ```bash
  kubectl get secret homelab-ca-key-pair -n cert-manager \
    -o jsonpath='{.data.tls\.crt}' | base64 -d > homelab-ca.crt
  ```
  Ohne Import zeigt der Browser eine Warnung – die Funktion (inkl.
  SharedArrayBuffer) ist davon nicht betroffen.

### cert-manager Versionshinweis

Der Cluster laeuft auf Kubernetes 1.29. cert-manager ist auf `v1.16.5` gepinnt,
weil neuere Versionen (ab 1.17) das CRD-Feld `selectableFields` nutzen, das erst
ab Kubernetes 1.30 existiert. Bei einem k8s-Upgrade auf >= 1.30 kann
cert-manager wieder angehoben werden.
