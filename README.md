# Supervision HP Z800 avec Prometheus + Grafana

Guide complet d'installation et de configuration de Prometheus et Grafana pour superviser une workstation HP Z800 sous Linux Server.

## Table des matières
1. [Prérequis](#prérequis)
2. [Installation de Prometheus](#installation-de-prometheus)
3. [Installation de Node Exporter](#installation-de-node-exporter)
4. [Installation de Grafana](#installation-de-grafana)
5. [Configuration](#configuration)
6. [Création des tableaux de bord](#création-des-tableaux-de-bord)
7. [Sécurisation](#sécurisation)
8. [Dépannage](#dépannage)

---

## Prérequis

- HP Z800 avec Linux Server installé (Ubuntu/Debian/CentOS/RHEL)
- Accès root ou sudo
- Connexion internet
- Ports disponibles : 9090 (Prometheus), 3000 (Grafana), 9100 (Node Exporter)

### Mise à jour du système
```bash
# Debian/Ubuntu
sudo apt update && sudo apt upgrade -y

# CentOS/RHEL
sudo yum update -y
```

---

## Installation de Prometheus

### 1. Créer un utilisateur système pour Prometheus
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

### 2. Télécharger Prometheus
```bash
cd /tmp
PROM_VERSION="2.48.0"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
```

### 3. Extraire et installer
```bash
tar -xvf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64

sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### 4. Créer le fichier de configuration
```bash
sudo nano /etc/prometheus/prometheus.yml
```

Contenu du fichier :
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
```

```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### 5. Créer le service systemd
```bash
sudo nano /etc/systemd/system/prometheus.service
```

Contenu :
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

### 6. Démarrer Prometheus
```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

### 7. Vérifier l'accès
Ouvrez un navigateur et accédez à : `http://IP_Z800:9090`

---

## Installation de Node Exporter

Node Exporter collecte les métriques système (CPU, RAM, disque, réseau).

### 1. Télécharger Node Exporter
```bash
cd /tmp
NODE_EXPORTER_VERSION="1.7.0"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
```

### 2. Installer
```bash
tar -xvf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo cp node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### 3. Créer le service systemd
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Contenu :
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

### 4. Démarrer Node Exporter
```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

### 5. Vérifier
```bash
curl http://localhost:9100/metrics
```

---

## Installation de Grafana

### Pour Ubuntu/Debian
```bash
# Installer les dépendances
sudo apt-get install -y software-properties-common wget

# Ajouter le repository Grafana
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Installer Grafana
sudo apt-get update
sudo apt-get install -y grafana
```

### Pour CentOS/RHEL
```bash
# Créer le fichier de repository
sudo nano /etc/yum.repos.d/grafana.repo
```

Contenu :
```ini
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

```bash
# Installer
sudo yum install -y grafana
```

### Démarrer Grafana
```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

### Accéder à Grafana
Ouvrez : `http://IP_Z800:3000`
- **Login par défaut** : admin
- **Mot de passe par défaut** : admin

(Vous serez invité à changer le mot de passe)

---

## Configuration

### 1. Ajouter Prometheus comme source de données dans Grafana

1. Connectez-vous à Grafana
2. Allez dans **Configuration** (icône engrenage) → **Data Sources**
3. Cliquez sur **Add data source**
4. Sélectionnez **Prometheus**
5. Configurez :
   - **URL** : `http://localhost:9090`
   - Laissez les autres paramètres par défaut
6. Cliquez sur **Save & Test**

### 2. Configurer le pare-feu (si actif)

```bash
# UFW (Ubuntu/Debian)
sudo ufw allow 9090/tcp comment 'Prometheus'
sudo ufw allow 3000/tcp comment 'Grafana'
sudo ufw allow 9100/tcp comment 'Node Exporter'

# Firewalld (CentOS/RHEL)
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```

---

## Création des tableaux de bord

### Option 1 : Importer un dashboard existant (Recommandé)

1. Dans Grafana, cliquez sur **+** → **Import**
2. Entrez l'ID du dashboard : **1860** (Node Exporter Full)
3. Cliquez sur **Load**
4. Sélectionnez votre source de données Prometheus
5. Cliquez sur **Import**

### Option 2 : Créer un dashboard personnalisé

1. Cliquez sur **+** → **Dashboard** → **Add new panel**
2. Exemples de requêtes PromQL :

**Utilisation CPU :**
```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Utilisation RAM :**
```promql
100 * (1 - ((node_memory_MemAvailable_bytes) / (node_memory_MemTotal_bytes)))
```

**Utilisation disque :**
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} * 100) / node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})
```

**Trafic réseau (envoi) :**
```promql
rate(node_network_transmit_bytes_total[5m])
```

**Trafic réseau (réception) :**
```promql
rate(node_network_receive_bytes_total[5m])
```

---

## Sécurisation

### 1. Changer le mot de passe admin Grafana

Lors de la première connexion, Grafana vous demandera de changer le mot de passe.

### 2. Configurer un reverse proxy avec Nginx (optionnel)

```bash
sudo apt install nginx -y  # ou sudo yum install nginx -y
```

```bash
sudo nano /etc/nginx/sites-available/grafana
```

Contenu :
```nginx
server {
    listen 80;
    server_name monitoring.votredomaine.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 3. Activer HTTPS avec Let's Encrypt (recommandé)

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d monitoring.votredomaine.com
```

### 4. Restreindre l'accès par IP (optionnel)

Modifier `/etc/prometheus/prometheus.yml` pour utiliser une authentification ou utilisez iptables :

```bash
# Autoriser seulement votre IP
sudo iptables -A INPUT -p tcp --dport 9090 -s VOTRE_IP -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9090 -j DROP
```

---

## Dépannage

### Vérifier l'état des services
```bash
sudo systemctl status prometheus
sudo systemctl status node_exporter
sudo systemctl status grafana-server
```

### Consulter les logs
```bash
sudo journalctl -u prometheus -f
sudo journalctl -u node_exporter -f
sudo journalctl -u grafana-server -f
```

### Tester la collecte de métriques
```bash
# Vérifier que Node Exporter fonctionne
curl http://localhost:9100/metrics

# Vérifier que Prometheus collecte les données
curl http://localhost:9090/api/v1/targets
```

### Problèmes courants

**Prometheus ne démarre pas :**
- Vérifier les permissions : `sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus`
- Vérifier la syntaxe du fichier de config : `/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml`

**Grafana ne se connecte pas à Prometheus :**
- Vérifier que Prometheus est accessible : `curl http://localhost:9090`
- Vérifier l'URL dans la configuration de la source de données

**Pas de données dans Grafana :**
- Vérifier que Node Exporter est en cours d'exécution
- Vérifier les targets dans Prometheus : `http://IP_Z800:9090/targets`

---

## Dashboards recommandés pour Grafana

Voici quelques IDs de dashboards populaires à importer :

- **1860** - Node Exporter Full (le plus complet)
- **405** - Node Exporter Server Metrics
- **11074** - Node Exporter for Prometheus Dashboard
- **12486** - Node Exporter Quickstart and Dashboard

---

## Maintenance

### Sauvegarde de la configuration

```bash
# Sauvegarder Prometheus
sudo tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz /etc/prometheus /var/lib/prometheus

# Sauvegarder Grafana
sudo tar -czf grafana-backup-$(date +%Y%m%d).tar.gz /etc/grafana /var/lib/grafana
```

### Mise à jour

```bash
# Grafana
sudo apt update && sudo apt upgrade grafana  # Debian/Ubuntu
sudo yum update grafana  # CentOS/RHEL

# Prometheus et Node Exporter : télécharger la nouvelle version et répéter les étapes d'installation
```

---

## Ressources supplémentaires

- [Documentation officielle Prometheus](https://prometheus.io/docs/)
- [Documentation officielle Grafana](https://grafana.com/docs/)
- [PromQL Queries](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Dashboard Library](https://grafana.com/grafana/dashboards/)

---

## Auteur

Guide créé pour la supervision d'une HP Z800 workstation sous Linux Server.

Date de création : Janvier 2026
