# PromQL Cheatsheet

Aide-mémoire des requêtes PromQL les plus utiles pour superviser votre HP Z800.

---

## 📖 Syntaxe de base

### Opérateurs mathématiques
```promql
+ - * / % ^    # Addition, soustraction, multiplication, division, modulo, puissance
```

### Opérateurs de comparaison
```promql
== != < > <= >=    # Égal, différent, inférieur, supérieur, etc.
```

### Fonctions temporelles
```promql
rate(metric[5m])        # Taux de changement par seconde sur 5 minutes
irate(metric[5m])       # Taux instantané (plus réactif)
increase(metric[5m])    # Augmentation totale sur 5 minutes
avg_over_time(metric[5m])  # Moyenne sur 5 minutes
```

### Agrégations
```promql
sum(metric)           # Somme
avg(metric)           # Moyenne
min(metric)           # Minimum
max(metric)           # Maximum
count(metric)         # Compte

# Avec groupement
sum by (label) (metric)        # Somme groupée par label
avg without (label) (metric)   # Moyenne sans ce label
```

---

## 🖥️ Métriques CPU

### Utilisation CPU globale (%)
```promql
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Utilisation par core (%)
```promql
100 - (irate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)
```

### Load average
```promql
node_load1    # 1 minute
node_load5    # 5 minutes
node_load15   # 15 minutes
```

### Temps CPU par mode
```promql
rate(node_cpu_seconds_total[5m])
# Modes disponibles: idle, system, user, iowait, irq, softirq
```

---

## 💾 Métriques Mémoire

### RAM utilisée (%)
```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

### RAM disponible (GB)
```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

### RAM utilisée (GB)
```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 1024 / 1024 / 1024
```

### SWAP utilisé (%)
```promql
100 - ((node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes) * 100)
```

### Cache et buffers (GB)
```promql
(node_memory_Cached_bytes + node_memory_Buffers_bytes) / 1024 / 1024 / 1024
```

---

## 💿 Métriques Disque

### Espace utilisé racine (%)
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} * 100) / 
        node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})
```

### Espace utilisé /mnt/data (%)
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/mnt/data"} * 100) / 
        node_filesystem_size_bytes{mountpoint="/mnt/data"})
```

### Tous les disques utilisés (%)
```promql
100 - ((node_filesystem_avail_bytes{fstype=~"ext4|xfs"} * 100) / 
        node_filesystem_size_bytes{fstype=~"ext4|xfs"})
```

### Espace disponible (GB)
```promql
node_filesystem_avail_bytes{mountpoint="/"} / 1024 / 1024 / 1024
```

### I/O Lecture (MB/s)
```promql
rate(node_disk_read_bytes_total[5m]) / 1024 / 1024
```

### I/O Écriture (MB/s)
```promql
rate(node_disk_written_bytes_total[5m]) / 1024 / 1024
```

### IOPS Lecture
```promql
rate(node_disk_reads_completed_total[5m])
```

### IOPS Écriture
```promql
rate(node_disk_writes_completed_total[5m])
```

---

## 🌐 Métriques Réseau

### Trafic reçu (Mbps)
```promql
rate(node_network_receive_bytes_total[5m]) * 8 / 1000000
```

### Trafic envoyé (Mbps)
```promql
rate(node_network_transmit_bytes_total[5m]) * 8 / 1000000
```

### Trafic total (Mbps)
```promql
(rate(node_network_receive_bytes_total[5m]) + 
 rate(node_network_transmit_bytes_total[5m])) * 8 / 1000000
```

### Paquets reçus par seconde
```promql
rate(node_network_receive_packets_total[5m])
```

### Erreurs réseau
```promql
rate(node_network_receive_errs_total[5m]) + 
rate(node_network_transmit_errs_total[5m])
```

---

## 🎮 Métriques GPU

### Température (°C)
```promql
nvidia_gpu_temperature_celsius
```

### Utilisation GPU (%)
```promql
nvidia_gpu_utilization
```

### Mémoire GPU utilisée (%)
```promql
(nvidia_gpu_memory_used_bytes / nvidia_gpu_memory_total_bytes) * 100
```

### Mémoire GPU utilisée (GB)
```promql
nvidia_gpu_memory_used_bytes / 1024 / 1024 / 1024
```

### Mémoire GPU disponible (GB)
```promql
(nvidia_gpu_memory_total_bytes - nvidia_gpu_memory_used_bytes) / 1024 / 1024 / 1024
```

### Consommation électrique (W)
```promql
nvidia_gpu_power_watts
```

### Alerte température élevée
```promql
nvidia_gpu_temperature_celsius > 75
```

---

## ⏱️ Métriques Système

### Uptime (jours)
```promql
(time() - node_boot_time_seconds) / 86400
```

### Uptime (heures)
```promql
(time() - node_boot_time_seconds) / 3600
```

### Nombre de CPU
```promql
count(node_cpu_seconds_total{mode="system"}) without (cpu, mode)
```

### Contexte switches par seconde
```promql
rate(node_context_switches_total[5m])
```

### Processus en cours
```promql
node_procs_running
```

---

## 🔔 Exemples d'alertes

### CPU > 80% pendant 5 minutes
```promql
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
```

### RAM > 90%
```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 90
```

### Disque > 85%
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / 
        node_filesystem_size_bytes{mountpoint="/"}) > 85
```

### GPU température > 80°C
```promql
nvidia_gpu_temperature_celsius > 80
```

### Service down
```promql
up == 0
```

---

## 📊 Requêtes avancées

### Top 5 processus CPU (nécessite process-exporter)
```promql
topk(5, rate(process_cpu_seconds_total[5m]))
```

### Prédiction espace disque (dans 4 heures)
```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600)
```

### Corrélation CPU et température GPU
```promql
# Dans deux panels côte à côte
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
nvidia_gpu_temperature_celsius
```

### Agrégation temporelle - Moyenne sur 1 heure
```promql
avg_over_time(nvidia_gpu_temperature_celsius[1h])
```

### Maximum GPU température sur 24h
```promql
max_over_time(nvidia_gpu_temperature_celsius[24h])
```

---

## 🎯 Bonnes pratiques

### 1. Choisir le bon intervalle
```promql
rate(metric[1m])     # Très réactif, plus de bruit
rate(metric[5m])     # Bon équilibre (recommandé)
rate(metric[15m])    # Lissé, moins réactif
```

### 2. Éviter les requêtes lourdes
```promql
# ❌ Mauvais (très lourd)
rate(node_cpu_seconds_total[24h])

# ✅ Bon (léger)
rate(node_cpu_seconds_total[5m])
```

### 3. Utiliser des labels pour filtrer
```promql
# Filtrer un device spécifique
node_filesystem_avail_bytes{device="/dev/sda1"}

# Regex
node_network_receive_bytes_total{device=~"eth.*"}

# Exclure
node_filesystem_avail_bytes{fstype!="tmpfs"}
```

### 4. Combiner plusieurs métriques
```promql
# Ratio lecture/écriture disque
rate(node_disk_read_bytes_total[5m]) / 
rate(node_disk_written_bytes_total[5m])
```

---

## 🔍 Debug et exploration

### Voir toutes les métriques disponibles
```promql
{__name__=~".+"}
```

### Voir toutes les métriques d'un job
```promql
{job="z800_node"}
```

### Voir les labels d'une métrique
```promql
node_cpu_seconds_total
# Regardez les labels dans le résultat
```

### Compter le nombre de séries temporelles
```promql
count({__name__=~".+"})
```

---

## 📝 Templates pour Grafana

### Variable pour mountpoint
```
label_values(node_filesystem_size_bytes, mountpoint)
```

### Variable pour device réseau
```
label_values(node_network_receive_bytes_total, device)
```

### Utiliser la variable dans une requête
```promql
node_filesystem_avail_bytes{mountpoint="$mountpoint"}
```

---

## 🆘 Aide

### Tester une requête
1. Allez sur Prometheus : http://192.168.1.108:9090
2. Onglet **Graph**
3. Collez votre requête
4. Cliquez sur **Execute**

### Voir la documentation d'une métrique
Dans Prometheus, survolez la métrique pour voir sa description.

### Ressources
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [PromQL Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [PromQL Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)

---

**💡 Astuce :** Sauvegardez vos requêtes favorites dans un fichier texte ou créez des panels Grafana réutilisables !
