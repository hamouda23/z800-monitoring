# Configurations - Prometheus & Grafana

Toutes les configurations de votre système de supervision HP Z800.

---

## 📋 Table des matières

1. [Architecture du système](#architecture-du-système)
2. [Configuration Prometheus](#configuration-prometheus)
3. [Configuration Node Exporter](#configuration-node-exporter)
4. [Configuration GPU Exporter](#configuration-gpu-exporter)
5. [Configuration Grafana](#configuration-grafana)
6. [Configuration Pare-feu](#configuration-pare-feu)
7. [Fichiers de service systemd](#fichiers-de-service-systemd)
8. [Dashboard GPU JSON](#dashboard-gpu-json)

---

## Architecture du système

### Infrastructure

```
┌─────────────────────┐         SSH/HTTP        ┌──────────────────────────┐
│  Laptop Windows 11  │ ────────────────────>   │   HP Z800 Workstation    │
│                     │                          │   Ubuntu Server 22.04    │
│  - Navigateur Web   │ <─ HTTP (3000, 9090)    │   IP: 192.168.1.108      │
│  - PowerShell/SSH   │                          │                          │
└─────────────────────┘                          │  Services:               │
                                                 │  - Prometheus   :9090    │
                                                 │  - Grafana      :3000    │
                                                 │  - Node Exporter:9100    │
                                                 │  - GPU Exporter :9835    │
                                                 └──────────────────────────┘
```

### Composants installés

| Composant | Version | Port | Chemin installation |
|-----------|---------|------|---------------------|
| **Prometheus** | 2.48.0 | 9090 | `/usr/local/bin/prometheus` |
| **Node Exporter** | 1.7.0 | 9100 | `/usr/local/bin/node_exporter` |
| **Grafana** | Latest | 3000 | Package APT |
| **GPU Exporter** | Custom Python | 9835 | `/usr/local/bin/simple-gpu-exporter.py` |

### Stockage des données

| Service | Données | Emplacement |
|---------|---------|-------------|
| **Prometheus** | Base de données TSDB | `/mnt/data/prometheus/data` (lien symbolique depuis `/var/lib/prometheus`) |
| **Grafana** | Configuration & dashboards | `/var/lib/grafana` |
| **Logs système** | Journaux systemd | `/var/log/journal` |

**Note importante** : Les données Prometheus ont été déplacées vers `/mnt/data` car la partition racine `/` était pleine.

---

## Configuration Prometheus

### Fichier de configuration principal

**Chemin :** `/etc/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'z800_node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'nvidia_gpu'
    static_configs:
      - targets: ['localhost:9835']
```

### Paramètres de configuration

- **scrape_interval** : 15s - Fréquence de collecte des métriques
- **evaluation_interval** : 15s - Fréquence d'évaluation des règles d'alerte
- **Rétention par défaut** : 15 jours

### Structure des dossiers

```
/etc/prometheus/
├── prometheus.yml           # Configuration principale
├── consoles/               # Templates de console
└── console_libraries/      # Bibliothèques de console

/var/lib/prometheus/        # Lien symbolique vers /mnt/data/prometheus/data
└── [données TSDB]

/mnt/data/prometheus/
└── data/                   # Données réelles de Prometheus
    ├── chunks_head/
    ├── wal/
    └── [autres fichiers TSDB]
```

### Commandes utiles

```bash
# Vérifier la configuration
/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# Voir les targets
curl http://localhost:9090/api/v1/targets | jq

# Tester une requête
curl 'http://localhost:9090/api/v1/query?query=up'

# Recharger la configuration (sans redémarrage)
curl -X POST http://localhost:9090/-/reload
```

### Modifier la rétention des données

Pour changer la durée de conservation (par défaut 15 jours) :

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Modifiez la ligne `ExecStart` :

```ini
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=30d \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
```

Puis :
```bash
sudo systemctl daemon-reload
sudo systemctl restart prometheus
```

---

## Configuration Node Exporter

### Installation

- **Binaire** : `/usr/local/bin/node_exporter`
- **Version** : 1.7.0
- **Port** : 9100

### Métriques collectées

Node Exporter collecte automatiquement :
- ✅ CPU (utilisation, fréquence, température)
- ✅ Mémoire (RAM, SWAP)
- ✅ Disques (utilisation, I/O)
- ✅ Réseau (trafic, erreurs)
- ✅ Système de fichiers
- ✅ Load average
- ✅ Uptime

### Pas de configuration nécessaire

Node Exporter fonctionne sans fichier de configuration. Il expose automatiquement toutes les métriques système disponibles.

### Vérifier les métriques

```bash
# Voir toutes les métriques
curl http://localhost:9100/metrics

# Filtrer les métriques CPU
curl http://localhost:9100/metrics | grep node_cpu

# Filtrer les métriques mémoire
curl http://localhost:9100/metrics | grep node_memory

# Filtrer les métriques disque
curl http://localhost:9100/metrics | grep node_filesystem
```

---

## Configuration GPU Exporter

### Script Python personnalisé

**Chemin :** `/usr/local/bin/simple-gpu-exporter.py`

```python
#!/usr/bin/env python3
import subprocess
import re
from http.server import HTTPServer, BaseHTTPRequestHandler
import time

class MetricsHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/metrics':
            self.send_response(200)
            self.send_header('Content-type', 'text/plain')
            self.end_headers()
            
            try:
                # Exécuter nvidia-smi
                result = subprocess.run([
                    'nvidia-smi',
                    '--query-gpu=temperature.gpu,utilization.gpu,utilization.memory,memory.used,memory.total,power.draw',
                    '--format=csv,noheader,nounits'
                ], capture_output=True, text=True, timeout=5)
                
                if result.returncode == 0:
                    data = result.stdout.strip().split(',')
                    temp = data[0].strip()
                    gpu_util = data[1].strip()
                    mem_util = data[2].strip()
                    mem_used = float(data[3].strip()) * 1024 * 1024  # MB to bytes
                    mem_total = float(data[4].strip()) * 1024 * 1024
                    power = data[5].strip()
                    
                    metrics = f"""# HELP nvidia_gpu_temperature_celsius GPU temperature in Celsius
# TYPE nvidia_gpu_temperature_celsius gauge
nvidia_gpu_temperature_celsius {temp}

# HELP nvidia_gpu_utilization GPU utilization percentage
# TYPE nvidia_gpu_utilization gauge
nvidia_gpu_utilization {gpu_util}

# HELP nvidia_gpu_memory_utilization GPU memory utilization percentage
# TYPE nvidia_gpu_memory_utilization gauge
nvidia_gpu_memory_utilization {mem_util}

# HELP nvidia_gpu_memory_used_bytes GPU memory used in bytes
# TYPE nvidia_gpu_memory_used_bytes gauge
nvidia_gpu_memory_used_bytes {mem_used}

# HELP nvidia_gpu_memory_total_bytes GPU memory total in bytes
# TYPE nvidia_gpu_memory_total_bytes gauge
nvidia_gpu_memory_total_bytes {mem_total}

# HELP nvidia_gpu_power_watts GPU power consumption in watts
# TYPE nvidia_gpu_power_watts gauge
nvidia_gpu_power_watts {power}
"""
                    self.wfile.write(metrics.encode())
                else:
                    self.wfile.write(b"# Error reading GPU metrics\n")
            except Exception as e:
                self.wfile.write(f"# Error: {e}\n".encode())
        else:
            self.send_response(404)
            self.end_headers()
    
    def log_message(self, format, *args):
        pass  # Désactiver les logs HTTP

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 9835), MetricsHandler)
    print('GPU Exporter running on port 9835...')
    server.serve_forever()
```

### Informations GPU

- **Modèle** : NVIDIA Quadro P4000
- **Mémoire** : 8 GB
- **Driver** : 580.126.09
- **CUDA** : 13.0

### Métriques exposées

| Métrique | Type | Description | Unité |
|----------|------|-------------|-------|
| `nvidia_gpu_temperature_celsius` | gauge | Température GPU | °C |
| `nvidia_gpu_utilization` | gauge | Utilisation GPU | % |
| `nvidia_gpu_memory_utilization` | gauge | Utilisation mémoire | % |
| `nvidia_gpu_memory_used_bytes` | gauge | Mémoire utilisée | bytes |
| `nvidia_gpu_memory_total_bytes` | gauge | Mémoire totale | bytes |
| `nvidia_gpu_power_watts` | gauge | Consommation | W |

### Tester le GPU Exporter

```bash
# Vérifier que nvidia-smi fonctionne
nvidia-smi

# Tester les métriques
curl http://localhost:9835/metrics

# Voir uniquement la température
curl http://localhost:9835/metrics | grep temperature
```

---

## Configuration Grafana

### Installation

- **Package** : Installé via APT depuis le repository officiel Grafana
- **Version** : Latest stable
- **Port** : 3000

### Source de données Prometheus

**Configuration dans Grafana :**

1. **Connections** → **Data Sources** → **Prometheus**
2. **URL** : `http://localhost:9090`
3. **Access** : Server (default)
4. **Scrape interval** : 15s

### Dashboards installés

| Dashboard | Description | Source |
|-----------|-------------|--------|
| **Node Exporter Full (ID: 1860)** | Supervision système complète | Grafana.com |
| **NVIDIA Quadro P4000 Monitoring** | Supervision GPU personnalisée | Custom JSON |

### Fichiers de configuration Grafana

```
/etc/grafana/
└── grafana.ini                 # Configuration principale

/var/lib/grafana/
├── grafana.db                  # Base de données SQLite
├── dashboards/                 # Dashboards sauvegardés
└── plugins/                    # Plugins installés

/var/log/grafana/
└── grafana.log                 # Logs Grafana
```

### Paramètres importants dans grafana.ini

```ini
[server]
http_port = 3000

[security]
admin_user = admin
admin_password = [crypté]

[database]
type = sqlite3
path = grafana.db

[analytics]
reporting_enabled = false
check_for_updates = true
```

### Accès à Grafana

- **URL** : http://192.168.1.108:3000
- **Login** : admin
- **Mot de passe** : [défini lors de la première connexion]

---

## Configuration Pare-feu

### UFW (Uncomplicated Firewall)

**Statut** : Actif

### Règles configurées

```bash
# Voir toutes les règles
sudo ufw status numbered
```

**Résultat :**

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 3000/tcp                   ALLOW IN    Anywhere                   # Grafana
[ 2] 22/tcp                     ALLOW IN    Anywhere                   # SSH
[ 3] 9090/tcp                   ALLOW IN    Anywhere                   # Prometheus
[ 4] 9100/tcp                   ALLOW IN    Anywhere                   # Node Exporter
[ 5] 9835/tcp                   ALLOW IN    Anywhere                   # GPU Exporter
```

### Commandes pare-feu

```bash
# Voir le statut
sudo ufw status verbose

# Activer le pare-feu
sudo ufw enable

# Ajouter une règle
sudo ufw allow 9090/tcp comment 'Prometheus'

# Supprimer une règle
sudo ufw delete [numéro]

# Désactiver le pare-feu (temporaire)
sudo ufw disable
```

### Ports réseau utilisés

| Port | Service | Description | Accès |
|------|---------|-------------|-------|
| **22** | SSH | Connexion à distance | Externe |
| **3000** | Grafana | Interface web | Externe |
| **9090** | Prometheus | Interface web + API | Externe |
| **9100** | Node Exporter | API métriques | Interne |
| **9835** | GPU Exporter | API métriques | Interne |

---

## Fichiers de service systemd

### Prometheus Service

**Chemin :** `/etc/systemd/system/prometheus.service`

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

### Node Exporter Service

**Chemin :** `/etc/systemd/system/node_exporter.service`

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

### GPU Exporter Service

**Chemin :** `/etc/systemd/system/simple-gpu-exporter.service`

```ini
[Unit]
Description=Simple GPU Exporter for Prometheus
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/simple-gpu-exporter.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Grafana Service

**Chemin :** `/lib/systemd/system/grafana-server.service` (installé via APT)

```ini
[Unit]
Description=Grafana instance
Documentation=http://docs.grafana.org
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=grafana
Group=grafana
ExecStart=/usr/sbin/grafana-server \
  --config=/etc/grafana/grafana.ini \
  --pidfile=/var/run/grafana/grafana-server.pid \
  --packaging=deb \
  cfg:default.paths.logs=/var/log/grafana \
  cfg:default.paths.data=/var/lib/grafana \
  cfg:default.paths.plugins=/var/lib/grafana/plugins \
  cfg:default.paths.provisioning=/etc/grafana/provisioning

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Commandes systemd

```bash
# Recharger les fichiers de service après modification
sudo systemctl daemon-reload

# Démarrer un service
sudo systemctl start [service]

# Arrêter un service
sudo systemctl stop [service]

# Redémarrer un service
sudo systemctl restart [service]

# Activer au démarrage
sudo systemctl enable [service]

# Voir le statut
sudo systemctl status [service]

# Voir les logs
sudo journalctl -u [service] -f
```

---

## Dashboard GPU JSON

Le dashboard GPU complet est disponible dans l'artifact "Dashboard Grafana - NVIDIA Quadro P4000".

### Import du dashboard

1. Copiez le contenu JSON
2. Dans Grafana : **Dashboards** → **New** → **Import**
3. Collez le JSON
4. Sélectionnez la source de données **prometheus**
5. Cliquez sur **Import**

### Panels inclus

1. **🌡️ GPU Temperature** (Gauge) - Température en temps réel
2. **⚡ GPU Utilization** (Gauge) - Utilisation GPU
3. **💾 GPU Memory Usage** (Gauge) - Utilisation mémoire
4. **🔋 GPU Power Consumption** (Time series) - Consommation électrique
5. **📈 GPU Temperature History** (Time series) - Historique température
6. **📊 GPU & Memory Utilization History** (Time series) - Historique utilisation
7. **Memory Used** (Stat) - Mémoire utilisée
8. **Total Memory** (Stat) - Mémoire totale

---

## Sauvegarde et restauration

### Sauvegarder la configuration Prometheus

```bash
# Créer une archive
sudo tar -czf prometheus-config-$(date +%Y%m%d).tar.gz \
  /etc/prometheus/prometheus.yml \
  /etc/systemd/system/prometheus.service

# Sauvegarder les données (attention, peut être volumineux)
sudo tar -czf prometheus-data-$(date +%Y%m%d).tar.gz /mnt/data/prometheus/data
```

### Sauvegarder Grafana

```bash
# Sauvegarder la configuration et les dashboards
sudo tar -czf grafana-backup-$(date +%Y%m%d).tar.gz \
  /etc/grafana \
  /var/lib/grafana
```

### Exporter les dashboards individuellement

Dans Grafana :
1. Ouvrez le dashboard
2. **⚙️ Settings** → **JSON Model**
3. Copiez le JSON
4. Sauvegardez dans un fichier `.json`

### Restaurer une configuration

```bash
# Restaurer Prometheus
sudo tar -xzf prometheus-config-YYYYMMDD.tar.gz -C /

# Restaurer Grafana
sudo tar -xzf grafana-backup-YYYYMMDD.tar.gz -C /
sudo systemctl restart grafana-server
```

---

## Variables d'environnement

Aucune variable d'environnement spécifique n'est requise. Tous les services utilisent leurs configurations par défaut.

---

## Utilisateurs système

### Utilisateurs créés pour les services

| Utilisateur | Service | Home | Shell | Permissions |
|-------------|---------|------|-------|-------------|
| **prometheus** | Prometheus | Aucun | /bin/false | Lecture/écriture sur `/var/lib/prometheus` et `/etc/prometheus` |
| **node_exporter** | Node Exporter | Aucun | /bin/false | Lecture système uniquement |
| **grafana** | Grafana | Aucun | /bin/false | Lecture/écriture sur `/var/lib/grafana` et `/etc/grafana` |

**Note** : Le GPU Exporter s'exécute en tant que root car il nécessite l'accès à `nvidia-smi`.

### Permissions des fichiers

```bash
# Prometheus
drwxr-xr-x prometheus:prometheus /etc/prometheus
drwxr-xr-x prometheus:prometheus /var/lib/prometheus
drwxr-xr-x prometheus:prometheus /mnt/data/prometheus

# Node Exporter
-rwxr-xr-x root:root /usr/local/bin/node_exporter

# GPU Exporter
-rwxr-xr-x root:root /usr/local/bin/simple-gpu-exporter.py

# Grafana
drwxr-xr-x grafana:grafana /etc/grafana
drwxr-xr-x grafana:grafana /var/lib/grafana
```

---

## Notes importantes

### Espace disque

⚠️ **Attention** : La partition racine `/` était pleine (100%). Les données Prometheus ont été déplacées vers `/mnt/data` qui dispose de 142 GB libres.

**Structure actuelle :**
```
/var/lib/prometheus → /mnt/data/prometheus/data (lien symbolique)
/mnt/data/prometheus/data (données réelles)
```

### Démarrage automatique

Tous les services sont configurés pour démarrer automatiquement au boot :

```bash
sudo systemctl is-enabled prometheus node_exporter grafana-server simple-gpu-exporter
```

Devrait retourner `enabled` pour chacun.

### Logs

Les logs sont gérés par systemd/journald :

```bash
# Voir les logs en temps réel
sudo journalctl -u prometheus -f
sudo journalctl -u node_exporter -f
sudo journalctl -u grafana-server -f
sudo journalctl -u simple-gpu-exporter -f

# Voir les 50 dernières lignes
sudo journalctl -u prometheus -n 50
```

---
## ⚠️ Problème connu : Clonage de disque

### Symptôme
Après un clonage de disque, snapshot ou déplacement :
- Prometheus démarre mais ne retourne aucune donnée
- Les targets sont UP
- Grafana affiche des dashboards vides
- `curl 'http://localhost:9090/api/v1/query?query=up'` retourne `"result":[]`

### Cause
La base de données TSDB de Prometheus est corrompue après le clonage. Les fichiers mmap ne supportent pas d'être copiés pendant que Prometheus tourne ou changent d'UUID de disque.

### Solution rapide (perte d'historique)
```bash
# Arrêter Prometheus
sudo systemctl stop prometheus

# Supprimer les données corrompues
sudo rm -rf /mnt/data/prometheus/data/*

# Recréer la structure
sudo mkdir -p /mnt/data/prometheus/data
sudo chown -R prometheus:prometheus /mnt/data/prometheus

# Redémarrer
sudo systemctl start prometheus

# Attendre 30 secondes
sleep 30

# Vérifier
curl 'http://localhost:9090/api/v1/query?query=up'
```

⚠️ **Note** : Vous perdez tout l'historique (données des 15 derniers jours)

### Solution avec sauvegarde de l'historique

Si vous voulez GARDER l'historique avant un clonage :
```bash
# AVANT le clonage/snapshot

# 1. Arrêter Prometheus proprement
sudo systemctl stop prometheus

# 2. Créer un snapshot Prometheus (optionnel)
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# 3. Faire votre clonage/snapshot du disque

# 4. APRÈS le clonage, sur le nouveau système :

# Arrêter Prometheus
sudo systemctl stop prometheus

# Supprimer les données
sudo rm -rf /mnt/data/prometheus/data/*

# Recréer
sudo mkdir -p /mnt/data/prometheus/data
sudo chown -R prometheus:prometheus /mnt/data/prometheus

# Redémarrer
sudo systemctl start prometheus
```

### Bonnes pratiques

✅ **TOUJOURS arrêter Prometheus avant un clonage** :
```bash
sudo systemctl stop prometheus
```

✅ **Sauvegarder la configuration** (pas les données TSDB) :
```bash
sudo tar -czf prometheus-config-$(date +%Y%m%d).tar.gz \
  /etc/prometheus/ \
  /etc/systemd/system/prometheus.service
```

✅ **Accepter la perte d'historique** : Les données TSDB ne sont pas faites pour être migrées. Après un clonage, recommencez la collecte.

✅ **Exporter les dashboards Grafana** (ils survivent au clonage) :
- Settings → JSON Model → Copier
- Sauvegarder dans le repository Git

❌ **Ne PAS sauvegarder** `/mnt/data/prometheus/data/` - ces fichiers ne sont pas portables

## Historique des modifications

### 2026-01-25 - Configuration initiale

- ✅ Installation Prometheus 2.48.0
- ✅ Installation Node Exporter 1.7.0
- ✅ Installation Grafana (latest)
- ✅ Création GPU Exporter personnalisé (Python)
- ✅ Configuration pare-feu UFW
- ✅ Déplacement données Prometheus vers `/mnt/data`
- ✅ Import dashboards Node Exporter et GPU
- ✅ Configuration complète et testée

---

**Configuration :** HP Z800 + Ubuntu Server 22.04 + Windows 11  
**Workstation IP :** 192.168.1.108  
**Dernière mise à jour :** Janvier 2026
