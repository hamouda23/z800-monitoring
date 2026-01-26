# Guide d'utilisation - Prometheus & Grafana

Guide pratique pour utiliser votre système de supervision HP Z800 avec Prometheus et Grafana.

---

## 📋 Table des matières

1. [Accès aux interfaces](#accès-aux-interfaces)
2. [Utilisation de Prometheus](#utilisation-de-prometheus)
3. [Utilisation de Grafana](#utilisation-de-grafana)
4. [Requêtes PromQL utiles](#requêtes-promql-utiles)
5. [Gestion des dashboards](#gestion-des-dashboards)
6. [Maintenance quotidienne](#maintenance-quotidienne)
7. [Dépannage](#dépannage)

---

## Accès aux interfaces

### Depuis Windows 11

Votre workstation HP Z800 est accessible à l'adresse : `192.168.1.108`

| Service | URL | Login | Description |
|---------|-----|-------|-------------|
| **Prometheus** | http://192.168.1.108:9090 | Aucun | Collecte et stockage des métriques |
| **Grafana** | http://192.168.1.108:3000 | admin / [votre_mot_de_passe] | Visualisation et dashboards |

### Via SSH depuis Windows

```powershell
# Connexion SSH
ssh samir@192.168.1.108

# Ou avec tunnel SSH (si pare-feu)
ssh -L 3000:localhost:3000 -L 9090:localhost:9090 samir@192.168.1.108
```

---

## Utilisation de Prometheus

### Interface web de Prometheus

**URL :** http://192.168.1.108:9090

### 1. Vérifier les targets (cibles de collecte)

1. Allez sur `http://192.168.1.108:9090/targets`
2. Vérifiez que toutes les cibles sont **UP** (vertes) :
   - ✅ `prometheus` (localhost:9090)
   - ✅ `z800_node` (localhost:9100) - Métriques système
   - ✅ `nvidia_gpu` (localhost:9835) - Métriques GPU

Si une cible est **DOWN** (rouge), voir la section [Dépannage](#dépannage).

### 2. Explorer les métriques

1. Allez sur `http://192.168.1.108:9090/graph`
2. Dans la barre de requête, tapez le début d'une métrique
3. L'auto-complétion vous proposera les métriques disponibles
4. Cliquez sur **Execute** pour voir les données
5. Passez en mode **Graph** pour visualiser l'historique

### 3. Métriques disponibles

#### Métriques Système (Node Exporter)
- `node_cpu_seconds_total` - Utilisation CPU
- `node_memory_MemAvailable_bytes` - Mémoire disponible
- `node_filesystem_avail_bytes` - Espace disque disponible
- `node_network_receive_bytes_total` - Trafic réseau reçu
- `node_network_transmit_bytes_total` - Trafic réseau envoyé

#### Métriques GPU (Custom Exporter)
- `nvidia_gpu_temperature_celsius` - Température GPU
- `nvidia_gpu_utilization` - Utilisation GPU (%)
- `nvidia_gpu_memory_utilization` - Utilisation mémoire GPU (%)
- `nvidia_gpu_memory_used_bytes` - Mémoire GPU utilisée
- `nvidia_gpu_memory_total_bytes` - Mémoire GPU totale
- `nvidia_gpu_power_watts` - Consommation électrique

---

## Utilisation de Grafana

### Connexion à Grafana

**URL :** http://192.168.1.108:3000

**Login :** admin  
**Mot de passe :** [votre mot de passe personnalisé]

### 1. Naviguer dans les dashboards

1. Cliquez sur **Dashboards** dans le menu de gauche (icône quatre carrés)
2. Vous verrez vos dashboards :
   - **Node Exporter Full** - Supervision système complète
   - **NVIDIA Quadro P4000 Monitoring** - Supervision GPU

3. Cliquez sur un dashboard pour l'ouvrir

### 2. Personnaliser l'affichage

#### Changer la période d'affichage

En haut à droite, cliquez sur la période (ex: "Last 24 hours") :
- **Last 5 minutes** - Temps réel
- **Last 15 minutes** - Court terme
- **Last 1 hour** - Supervision normale
- **Last 24 hours** - Vue d'ensemble journalière
- **Custom** - Période personnalisée

#### Rafraîchissement automatique

Cliquez sur l'icône **Refresh** (🔄) en haut à droite :
- **5s** - Rafraîchissement toutes les 5 secondes (temps réel)
- **10s** - Toutes les 10 secondes
- **30s** - Toutes les 30 secondes
- **1m** - Toutes les minutes

### 3. Modifier un panel

1. Survolez le titre d'un panel
2. Cliquez sur le titre → **Edit**
3. Modifiez la requête ou les paramètres
4. Cliquez sur **Apply** pour sauvegarder

### 4. Ajouter un nouveau panel

1. Cliquez sur **+ Add** en haut à droite
2. Sélectionnez **Visualization**
3. Choisissez votre source de données : **prometheus**
4. Entrez votre requête PromQL
5. Choisissez le type de visualisation (Gauge, Time series, Table, etc.)
6. Configurez les options (titre, unité, seuils, etc.)
7. Cliquez sur **Apply**
8. **Save dashboard** pour sauvegarder

### 5. Exporter / Partager un dashboard

#### Exporter en JSON
1. Ouvrez le dashboard
2. Cliquez sur **⚙️** (Settings) en haut à droite
3. Allez dans **JSON Model**
4. Copiez le JSON ou cliquez sur **Save JSON to file**

#### Créer un snapshot (capture)
1. Cliquez sur **Share** (icône partage)
2. Onglet **Snapshot**
3. Cliquez sur **Publish snapshot**

---

## Requêtes PromQL utiles

### Métriques CPU

```promql
# Utilisation CPU en % (moyenne tous cores)
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Utilisation par core
100 - (irate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)

# Load average
node_load1
node_load5
node_load15
```

### Métriques Mémoire

```promql
# Utilisation RAM en %
100 * (1 - ((node_memory_MemAvailable_bytes) / (node_memory_MemTotal_bytes)))

# RAM disponible en GB
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# RAM utilisée en GB
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 1024 / 1024 / 1024

# SWAP utilisé en %
100 - ((node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes) * 100)
```

### Métriques Disque

```promql
# Utilisation disque racine en %
100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} * 100) / node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})

# Utilisation de tous les disques
100 - ((node_filesystem_avail_bytes{fstype=~"ext4|xfs"} * 100) / node_filesystem_size_bytes{fstype=~"ext4|xfs"})

# Espace disponible en GB
node_filesystem_avail_bytes{mountpoint="/"} / 1024 / 1024 / 1024

# I/O disque - Lecture (bytes/sec)
rate(node_disk_read_bytes_total[5m])

# I/O disque - Écriture (bytes/sec)
rate(node_disk_written_bytes_total[5m])
```

### Métriques Réseau

```promql
# Trafic réseau reçu (bytes/sec)
rate(node_network_receive_bytes_total[5m])

# Trafic réseau envoyé (bytes/sec)
rate(node_network_transmit_bytes_total[5m])

# Trafic total (reçu + envoyé) en Mbps
(rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])) * 8 / 1000000
```

### Métriques GPU

```promql
# Température GPU
nvidia_gpu_temperature_celsius

# Utilisation GPU
nvidia_gpu_utilization

# Mémoire GPU utilisée en %
(nvidia_gpu_memory_used_bytes / nvidia_gpu_memory_total_bytes) * 100

# Mémoire GPU utilisée en GB
nvidia_gpu_memory_used_bytes / 1024 / 1024 / 1024

# Consommation électrique
nvidia_gpu_power_watts
```

### Uptime système

```promql
# Uptime en jours
(time() - node_boot_time_seconds) / 86400

# Uptime en heures
(time() - node_boot_time_seconds) / 3600
```

---

## Gestion des dashboards

### Importer un dashboard depuis Grafana.com

1. Trouvez un dashboard sur https://grafana.com/grafana/dashboards/
2. Notez l'ID du dashboard (ex: 1860, 12486, 14574)
3. Dans Grafana, allez dans **Dashboards** → **New** → **Import**
4. Entrez l'ID du dashboard
5. Cliquez sur **Load**
6. Sélectionnez votre source de données **prometheus**
7. Cliquez sur **Import**

### Dashboards recommandés

| ID | Nom | Description |
|----|-----|-------------|
| **1860** | Node Exporter Full | Dashboard système complet (installé) |
| **12486** | Node Exporter Quickstart | Alternative avec plus de détails |
| **11074** | Node Exporter for Prometheus | Autre alternative populaire |

### Sauvegarder vos dashboards

Les dashboards sont automatiquement sauvegardés dans Grafana, mais pour une sauvegarde externe :

1. Ouvrez le dashboard
2. **⚙️ Settings** → **JSON Model**
3. Copiez le JSON et sauvegardez-le dans un fichier `.json`
4. Commitez le fichier dans votre repository GitHub

### Organiser les dashboards

1. Créez des **dossiers** : **Dashboards** → **New Folder**
2. Déplacez les dashboards dans les dossiers
3. Ajoutez des **tags** pour faciliter la recherche

---

## Maintenance quotidienne

### Vérifications rapides (5 minutes)

1. **Vérifier que les services tournent** :
   ```bash
   sudo systemctl status prometheus node_exporter grafana-server simple-gpu-exporter
   ```

2. **Vérifier l'espace disque** :
   ```bash
   df -h /
   ```
   ⚠️ Alerte si > 90% utilisé

3. **Vérifier les targets Prometheus** :
   - Allez sur http://192.168.1.108:9090/targets
   - Toutes doivent être **UP**

4. **Vérifier Grafana** :
   - Ouvrez http://192.168.1.108:3000
   - Vérifiez que les dashboards affichent des données

### Maintenance hebdomadaire (10 minutes)

1. **Nettoyer les logs** :
   ```bash
   sudo journalctl --vacuum-time=7d
   ```

2. **Vérifier les mises à jour** :
   ```bash
   sudo apt update
   sudo apt list --upgradable
   ```

3. **Exporter les dashboards** :
   - Sauvegardez les dashboards modifiés en JSON
   - Commitez dans GitHub

### Maintenance mensuelle (30 minutes)

1. **Mettre à jour Grafana** :
   ```bash
   sudo apt update
   sudo apt upgrade grafana
   sudo systemctl restart grafana-server
   ```

2. **Vérifier l'espace Prometheus** :
   ```bash
   du -sh /var/lib/prometheus
   ```
   Si > 10GB, considérez réduire la rétention

3. **Nettoyer les anciennes données** :
   ```bash
   # Nettoyer les packages inutiles
   sudo apt autoremove -y
   sudo apt clean
   ```

---

## Dépannage

### Prometheus ne démarre pas

```bash
# Vérifier le statut
sudo systemctl status prometheus

# Voir les logs
sudo journalctl -u prometheus -n 50 --no-pager

# Vérifier la configuration
/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# Redémarrer
sudo systemctl restart prometheus
```

### Grafana ne démarre pas

```bash
# Vérifier le statut
sudo systemctl status grafana-server

# Voir les logs
sudo journalctl -u grafana-server -n 50 --no-pager

# Redémarrer
sudo systemctl restart grafana-server
```

### GPU Exporter ne fonctionne pas

```bash
# Vérifier le statut
sudo systemctl status simple-gpu-exporter

# Tester nvidia-smi manuellement
nvidia-smi

# Tester les métriques
curl http://localhost:9835/metrics

# Redémarrer
sudo systemctl restart simple-gpu-exporter
```

### Une target est DOWN dans Prometheus

1. Identifiez quelle target est DOWN
2. Vérifiez le service correspondant :
   ```bash
   sudo systemctl status [service_name]
   ```
3. Redémarrez le service :
   ```bash
   sudo systemctl restart [service_name]
   ```

### Grafana ne se connecte pas à Prometheus

1. Allez dans **Connections** → **Data Sources**
2. Cliquez sur **prometheus**
3. Vérifiez l'URL : `http://localhost:9090`
4. Cliquez sur **Save & Test**
5. Si erreur, vérifiez que Prometheus tourne

### Pas de données dans les dashboards

1. Vérifiez que Prometheus collecte des données :
   - Allez sur http://192.168.1.108:9090/targets
   - Toutes les targets doivent être UP

2. Testez une requête simple dans Prometheus :
   ```promql
   up
   ```
   Devrait retourner 1 pour chaque service

3. Vérifiez la période d'affichage dans Grafana
   - Changez pour "Last 5 minutes"

### Redémarrer tous les services

```bash
sudo systemctl restart prometheus node_exporter grafana-server simple-gpu-exporter
```

### Voir tous les ports en écoute

```bash
sudo ss -tulpn | grep -E '9090|9100|9835|3000'
```

Devrait afficher :
- Port 9090 - Prometheus
- Port 9100 - Node Exporter
- Port 9835 - GPU Exporter
- Port 3000 - Grafana

---

## Commandes utiles

### Gestion des services

```bash
# Démarrer
sudo systemctl start [service]

# Arrêter
sudo systemctl stop [service]

# Redémarrer
sudo systemctl restart [service]

# Activer au démarrage
sudo systemctl enable [service]

# Désactiver au démarrage
sudo systemctl disable [service]

# Voir le statut
sudo systemctl status [service]

# Voir les logs
sudo journalctl -u [service] -f
```

### Vérification de l'espace disque

```bash
# Espace disque global
df -h

# Espace utilisé par Prometheus
du -sh /var/lib/prometheus

# Espace utilisé par Grafana
du -sh /var/lib/grafana

# Trouver les gros fichiers
sudo du -h / 2>/dev/null | grep '^[0-9.]*G' | sort -h
```

### Nettoyage

```bash
# Nettoyer les logs anciens
sudo journalctl --vacuum-time=7d

# Nettoyer APT
sudo apt clean
sudo apt autoremove -y

# Nettoyer /tmp
sudo rm -rf /tmp/*
```

---

## Ressources supplémentaires

- **Documentation Prometheus** : https://prometheus.io/docs/
- **Documentation Grafana** : https://grafana.com/docs/
- **PromQL Guide** : https://prometheus.io/docs/prometheus/latest/querying/basics/
- **Grafana Dashboards** : https://grafana.com/grafana/dashboards/
- **Support Grafana** : https://community.grafana.com/

---

## Notes importantes

⚠️ **Sécurité** :
- Changez le mot de passe admin de Grafana
- Ne jamais exposer Prometheus/Grafana directement sur Internet sans authentification
- Utilisez un VPN ou SSH tunnel pour l'accès à distance

💡 **Performance** :
- Prometheus stocke ~2 semaines de données par défaut
- Chaque métrique consomme ~1-2 bytes/sample
- Avec 3 exporters, comptez ~100-200 MB/jour

🔧 **Personnalisation** :
- Tous les dashboards peuvent être modifiés
- Créez vos propres panels selon vos besoins
- Exportez et partagez vos dashboards personnalisés

---

**Dernière mise à jour :** Janvier 2026  
**Configuration :** HP Z800 + Ubuntu Server 22.04 + Windows 11
