# 🖥️ Supervision HP Z800 - Prometheus + Grafana

Système de monitoring complet pour workstation HP Z800 sous Ubuntu Server 22.04, accessible depuis Windows 11.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-orange.svg)
![Prometheus](https://img.shields.io/badge/Prometheus-2.48.0-red.svg)
![Grafana](https://img.shields.io/badge/Grafana-Latest-orange.svg)

---

## 📊 Vue d'ensemble

Ce projet fournit une solution complète de monitoring pour superviser :
- ✅ **CPU, RAM, Disques** - Métriques système complètes via Node Exporter
- ✅ **GPU NVIDIA Quadro P4000** - Température, utilisation, mémoire, consommation
- ✅ **Réseau** - Trafic entrant/sortant
- ✅ **Dashboards Grafana** - Visualisation temps réel avec alertes

### Architecture
```
┌─────────────────────┐         SSH          ┌──────────────────────────┐
│  Laptop Windows 11  │ ───────────────────> │   HP Z800 Workstation    │
│                     │                       │   Ubuntu Server 22.04    │
│  - Navigateur Web   │ <─ HTTP (3000,9090)  │                          │
│  - Terminal SSH     │                       │  - Prometheus (9090)     │
│    (PowerShell/     │                       │  - Grafana (3000)        │
│     PuTTY)          │                       │  - Node Exporter (9100)  │
└─────────────────────┘                       └──────────────────────────┘
```

---

```
Windows 11 Laptop  ──SSH/HTTP──>  HP Z800 (Ubuntu 22.04)
    │                                  │
    │                                  ├─ Prometheus :9090
    │                                  ├─ Grafana :3000
    └──── Navigateur Web ────>         ├─ Node Exporter :9100
          (dashboards)                 └─ GPU Exporter :9835
```

### 🎯 Fonctionnalités

- **Monitoring temps réel** avec rafraîchissement 5-15 secondes
- **Historique 15 jours** des métriques
- **Dashboards personnalisables** Grafana
- **Alertes configurables** (email, Slack, webhook)
- **Accès distant sécurisé** via SSH ou VPN
- **API REST** pour intégration avec d'autres outils

---

## 🚀 Quick Start

### Prérequis

- HP Z800 avec Ubuntu Server 22.04
- GPU NVIDIA avec drivers installés
- Connexion réseau entre Windows et HP Z800
- 2 GB espace disque libre minimum

### Installation rapide (15 minutes)

```bash
# 1. Cloner le repository
git clone https://github.com/votre-username/z800-monitoring.git
cd z800-monitoring

# 2. Lancer le script d'installation (à créer)
chmod +x scripts/install.sh
sudo ./scripts/install.sh

# 3. Vérifier que tout fonctionne
sudo systemctl status prometheus grafana-server node_exporter simple-gpu-exporter
```

### Accès aux interfaces

| Service | URL | Login |
|---------|-----|-------|
| **Grafana** | http://192.168.1.108:3000 | admin / [mot_de_passe] |
| **Prometheus** | http://192.168.1.108:9090 | Aucun |

---

## 📚 Documentation

### Pour les utilisateurs

- **[Guide de compréhension](docs/UNDERSTANDING.md)** - Comprendre Prometheus et Grafana
- **[Guide d'installation](docs/INSTALLATION.md)** - Installation pas à pas détaillée
- **[Guide d'utilisation](docs/USAGE.md)** - Comment utiliser Prometheus et Grafana
- **[PromQL Cheatsheet](docs/PROMQL_CHEATSHEET.md)** - Requêtes utiles pour interroger vos métriques

### Pour les administrateurs

- **[Configuration](docs/CONFIGURATION.md)** - Fichiers de config et architecture


### Concepts clés

#### Qu'est-ce que Prometheus ?

Prometheus est une **base de données de métriques** (TSDB - Time Series Database) qui :
1. **Collecte** des métriques depuis vos services (CPU, RAM, GPU, etc.)
2. **Stocke** ces données avec un timestamp
3. **Permet de les interroger** avec le langage PromQL
4. **Génère des alertes** quand des seuils sont dépassés

**Exemple concret :** Toutes les 15 secondes, Prometheus interroge Node Exporter pour récupérer l'utilisation CPU, et stocke ces valeurs dans sa base de données.

#### Qu'est-ce que Grafana ?

Grafana est un **outil de visualisation** qui :
1. **Se connecte** à Prometheus pour lire les métriques
2. **Affiche** les données sous forme de graphiques, jauges, tableaux
3. **Crée des dashboards** personnalisables
4. **Permet d'explorer** vos données de façon interactive

**Exemple concret :** Grafana affiche un graphique montrant l'évolution de votre température GPU sur les dernières 24 heures, en interrogeant Prometheus avec la requête `nvidia_gpu_temperature_celsius`.

#### Comment ça fonctionne ensemble ?

```
1. Node Exporter lit l'utilisation CPU du système
2. Prometheus interroge Node Exporter toutes les 15s : "Quelle est l'utilisation CPU ?"
3. Prometheus stocke : "À 14h35:15, CPU = 45%"
4. Grafana demande à Prometheus : "Donne-moi l'historique CPU des 24 dernières heures"
5. Grafana affiche un beau graphique pour vous
```

---

## 📦 Composants installés

| Composant | Version | Rôle |
|-----------|---------|------|
| **Prometheus** | 2.48.0 | Collecte et stockage des métriques |
| **Node Exporter** | 1.7.0 | Expose les métriques système (CPU, RAM, disque) |
| **GPU Exporter** | Custom Python | Expose les métriques GPU NVIDIA |
| **Grafana** | Latest | Visualisation des dashboards |

---

## 🎨 Dashboards disponibles

### 1. Node Exporter Full (ID: 1860)
Dashboard complet pour la supervision système :
- Utilisation CPU par core
- Mémoire RAM et SWAP
- Espace disque et I/O
- Trafic réseau
- Load average et uptime

### 2. NVIDIA Quadro P4000 Monitoring
Dashboard personnalisé pour le GPU :
- 🌡️ Température en temps réel (gauge avec seuils colorés)
- ⚡ Utilisation GPU (%)
- 💾 Utilisation mémoire GPU (%)
- 🔋 Consommation électrique (W)
- 📈 Historiques sur 24h

**Import :** Le fichier JSON est dans `dashboards/nvidia-gpu-monitoring.json`

---

## 🔧 Configuration rapide

### Modifier les targets Prometheus

Éditez `/etc/prometheus/prometheus.yml` :

```yaml
scrape_configs:
  - job_name: 'z800_node'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'nvidia_gpu'
    static_configs:
      - targets: ['localhost:9835']
```

Redémarrez : `sudo systemctl restart prometheus`

### Ajouter un dashboard dans Grafana

1. Allez dans **Dashboards** → **New** → **Import**
2. Collez le contenu de `dashboards/nvidia-gpu-monitoring.json`
3. Sélectionnez la source **prometheus**
4. Cliquez sur **Import**

### Requêtes PromQL courantes

```promql
# Utilisation CPU moyenne
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Utilisation RAM (%)
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))

# Température GPU
nvidia_gpu_temperature_celsius

# Mémoire GPU utilisée (%)
(nvidia_gpu_memory_used_bytes / nvidia_gpu_memory_total_bytes) * 100
```

**Plus de requêtes :** Voir [PromQL Cheatsheet](docs/PROMQL_CHEATSHEET.md)

---

## 🛠️ Maintenance

### Vérification quotidienne (2 minutes)

```bash
# Vérifier que tous les services tournent
./scripts/check-services.sh

# Voir l'espace disque
df -h /mnt/data
```

### Sauvegarde hebdomadaire (5 minutes)

```bash
# Sauvegarder les configurations et dashboards
./scripts/backup.sh
```

### Commandes utiles

```bash
# Redémarrer tous les services
sudo systemctl restart prometheus node_exporter grafana-server simple-gpu-exporter

# Voir les logs en temps réel
sudo journalctl -u prometheus -f

# Vérifier les targets Prometheus
curl http://localhost:9090/api/v1/targets | jq
```

---

## 🐛 Problèmes courants

### Les services ne démarrent pas

```bash
# Vérifier le statut
sudo systemctl status prometheus

# Voir les erreurs détaillées
sudo journalctl -u prometheus -n 50
```

### Grafana ne se connecte pas à Prometheus

1. Vérifiez que Prometheus tourne : `curl http://localhost:9090`
2. Dans Grafana : **Connections** → **Data Sources** → **Prometheus**
3. URL doit être : `http://localhost:9090`
4. Cliquez sur **Save & Test**

### Pas de données GPU

```bash
# Vérifier nvidia-smi
nvidia-smi

# Vérifier le GPU Exporter
curl http://localhost:9835/metrics

# Redémarrer le service
sudo systemctl restart simple-gpu-exporter
```

**Plus de solutions :** Voir [Dépannage complet](docs/CONFIGURATION.md#dépannage)

---

## 📊 Captures d'écran

### Dashboard Système
**[[Node Exporter Dashboard](images/grafana-cpu.png)** - Node Exporter Dashboard

### Dashboard GPU
**[[Node Exporter Dashboard](images/grafana-gpu.png)** - Node Exporter Dashboard**

---

## 🤝 Contribution

Les contributions sont les bienvenues ! Pour contribuer :

1. Forkez le projet
2. Créez une branche (`git checkout -b feature/amelioration`)
3. Committez vos changements (`git commit -am 'Ajout fonctionnalité'`)
4. Poussez vers la branche (`git push origin feature/amelioration`)
5. Ouvrez une Pull Request

### Idées d'amélioration

- [ ] Script d'installation automatique complet
- [ ] Configuration d'alertes (email, Slack)
- [ ] Dashboard pour monitoring Docker (si applicable)
- [ ] Support multi-GPU
- [ ] Dashboard mobile-friendly
- [ ] Export automatique des métriques vers cloud

---

## 📝 Changelog

### Version 1.0.0 (2026-01-25)
- ✅ Installation initiale Prometheus + Grafana
- ✅ Configuration Node Exporter
- ✅ Développement GPU Exporter personnalisé (Python)
- ✅ Création dashboards GPU et système
- ✅ Documentation complète
- ✅ Déplacement données Prometheus vers `/mnt/data` (partition pleine)

---

## 📄 License

MIT License - Vous êtes libre d'utiliser, modifier et distribuer ce projet.

---

## 🔗 Ressources utiles

### Documentation officielle
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [PromQL Guide](https://prometheus.io/docs/prometheus/latest/querying/basics/)

### Dashboards Grafana
- [Grafana Dashboard Library](https://grafana.com/grafana/dashboards/)
- [Node Exporter Dashboards](https://grafana.com/grafana/dashboards/?search=node+exporter)

### Communauté
- [Prometheus Community](https://prometheus.io/community/)
- [Grafana Community](https://community.grafana.com/)

---

## 👤 Auteur

**Samir**
- Configuration : HP Z800 + Ubuntu Server 22.04
- Client : Windows 11
- GPU : NVIDIA Quadro P4000

---

## 🙏 Remerciements

- Projet Prometheus pour l'excellente TSDB
- Équipe Grafana pour l'outil de visualisation
- Communauté Node Exporter
- Documentation Ubuntu Server

---

**⭐ Si ce projet vous a été utile, n'hésitez pas à lui donner une étoile !**




