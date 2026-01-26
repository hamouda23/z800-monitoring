# Comprendre Prometheus et Grafana

Guide simple pour comprendre ce qu'est Prometheus et Grafana, comment ils fonctionnent ensemble, et comment les utiliser pour superviser votre HP Z800.

---

## 📖 Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Qu'est-ce que Prometheus ?](#quest-ce-que-prometheus)
3. [Qu'est-ce que Grafana ?](#quest-ce-que-grafana)
4. [Comment ils fonctionnent ensemble](#comment-ils-fonctionnent-ensemble)
5. [Utilisation pratique](#utilisation-pratique)
6. [Exemples concrets](#exemples-concrets)
7. [FAQ](#faq)

---

## Vue d'ensemble

### L'analogie de la météo 🌤️

Imaginez que vous voulez suivre la météo chez vous :

| Élément | Équivalent monitoring |
|---------|---------------------|
| **Thermomètre** | Node Exporter / GPU Exporter |
| **Cahier de notes** | Prometheus |
| **Graphiques muraux** | Grafana |

1. Le **thermomètre** mesure la température toutes les 15 secondes
2. Vous notez ces valeurs dans un **cahier** avec l'heure exacte
3. Vous créez de beaux **graphiques** pour visualiser l'évolution

**C'est exactement comme ça que fonctionne notre système de monitoring !**

---

## Qu'est-ce que Prometheus ?

### 🎯 Rôle : La base de données qui se souvient de tout

Prometheus est une **base de données spécialisée** dans le stockage de métriques avec leur horodatage.

### Comment ça marche ?

```
Toutes les 15 secondes, Prometheus demande :
┌─────────────────────────────────────────────────┐
│ "Hé Node Exporter, c'est quoi l'utilisation    │
│  CPU en ce moment ?"                            │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│ Node Exporter répond : "45%"                    │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│ Prometheus stocke :                             │
│ 2026-01-25 14h35:15 → CPU = 45%                │
│ 2026-01-25 14h35:30 → CPU = 47%                │
│ 2026-01-25 14h35:45 → CPU = 44%                │
│ ...                                             │
└─────────────────────────────────────────────────┘
```

### Ce que fait Prometheus

✅ **Collecte** des métriques depuis vos services (pull model)  
✅ **Stocke** ces données dans une base de données temporelle (TSDB)  
✅ **Permet de les interroger** avec le langage PromQL  
✅ **Génère des alertes** quand quelque chose ne va pas  
✅ **Garde l'historique** (par défaut 15 jours)

### Ce que Prometheus ne fait PAS

❌ N'affiche pas de jolis graphiques (c'est le rôle de Grafana)  
❌ Ne modifie pas votre système  
❌ N'est pas un tableau de bord

### Interface web de Prometheus

Vous pouvez accéder à Prometheus sur : `http://192.168.1.108:9090`

**À quoi ça sert ?**
- 🔍 Tester des requêtes PromQL
- 📊 Voir des graphiques basiques
- 🎯 Vérifier que les services sont bien collectés (targets)
- 🔧 Débugger des problèmes

---

## Qu'est-ce que Grafana ?

### 🎨 Rôle : La vitrine qui rend tout beau

Grafana est un **outil de visualisation** qui transforme vos données en beaux tableaux de bord.

### L'analogie du restaurant 🍽️

```
Prometheus = La cuisine (où tout est préparé)
Grafana    = La salle à manger (où tout est joliment présenté)

Vous ne mangez pas dans la cuisine, même si c'est là 
que la nourriture est préparée !
```

### Comment ça marche ?

```
1. Vous ouvrez Grafana dans votre navigateur
                ↓
2. Grafana demande à Prometheus :
   "Donne-moi l'utilisation CPU des dernières 24h"
                ↓
3. Prometheus répond avec toutes les données
                ↓
4. Grafana dessine un beau graphique
                ↓
5. Vous voyez : 📈 Un joli graphique coloré !
```

### Ce que fait Grafana

✅ **Se connecte** à Prometheus pour lire les métriques  
✅ **Crée des dashboards** personnalisables et interactifs  
✅ **Affiche** des graphiques, jauges, tableaux, cartes  
✅ **Permet d'explorer** vos données (zoomer, filtrer, etc.)  
✅ **Envoie des alertes** (email, Slack, etc.)  
✅ **Partage** des dashboards avec votre équipe

### Ce que Grafana ne fait PAS

❌ Ne stocke pas les données (il les lit depuis Prometheus)  
❌ Ne collecte pas de métriques  
❌ Ne fonctionne pas sans source de données

### Interface web de Grafana

Vous pouvez accéder à Grafana sur : `http://192.168.1.108:3000`

**À quoi ça sert ?**
- 📊 Voir vos métriques en temps réel
- 🎨 Créer des dashboards personnalisés
- 👀 Surveiller votre système d'un coup d'œil
- 📱 Accéder depuis n'importe quel navigateur

---

## Comment ils fonctionnent ensemble

### Le flux complet de données

```
┌──────────────────┐
│  Votre système   │
│  (HP Z800)       │
└────────┬─────────┘
         │
         │ 1. Expose les métriques
         ↓
┌──────────────────┐     ┌──────────────────┐
│  Node Exporter   │     │  GPU Exporter    │
│  (port 9100)     │     │  (port 9835)     │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
         │ 2. Collecte toutes les 15s
         └────────────┬───────────┘
                      ↓
              ┌──────────────┐
              │  Prometheus  │
              │  (port 9090) │
              │              │
              │  [Base de    │
              │   données]   │
              └──────┬───────┘
                     │
                     │ 3. Lit les données
                     ↓
              ┌──────────────┐
              │   Grafana    │
              │  (port 3000) │
              │              │
              │  [Affiche    │
              │   graphiques]│
              └──────┬───────┘
                     │
                     │ 4. Vous consultez
                     ↓
              ┌──────────────┐
              │  Navigateur  │
              │   Windows    │
              └──────────────┘
```

### Exemple concret : Surveiller la température GPU

**Étape 1 : GPU Exporter expose la métrique**
```bash
# Toutes les secondes, prêt à répondre
nvidia_gpu_temperature_celsius 45
```

**Étape 2 : Prometheus collecte (toutes les 15s)**
```
14:35:00 → 45°C
14:35:15 → 46°C
14:35:30 → 45°C
14:35:45 → 47°C
```

**Étape 3 : Grafana demande à Prometheus**
```promql
"Donne-moi nvidia_gpu_temperature_celsius des 24 dernières heures"
```

**Étape 4 : Grafana affiche**
```
    50°C ┤     ╭─╮
    45°C ┤ ╭───╯ ╰──╮
    40°C ┤─╯        ╰─
         └─────────────
         0h    12h   24h
```

---

## Utilisation pratique

### Scénario 1 : "Mon PC rame, pourquoi ?"

**Avec Prometheus + Grafana :**

1. Ouvrez Grafana : `http://192.168.1.108:3000`
2. Cliquez sur le dashboard **Node Exporter Full**
3. Regardez les panels :
   - 🔴 CPU à 95% ? → Un processus consomme trop
   - 🔴 RAM à 98% ? → Pas assez de mémoire
   - 🔴 Disque à 99% ? → Plus d'espace
   - 🟢 Tout est normal ? → Le problème est ailleurs

**Sans monitoring :** Vous devriez vérifier manuellement avec `top`, `free -h`, `df -h`, etc.

### Scénario 2 : "Le GPU chauffe-t-il trop ?"

**Avec Prometheus + Grafana :**

1. Ouvrez le dashboard **GPU Monitoring**
2. Regardez la température :
   - 🟢 < 60°C : Excellent
   - 🟡 60-75°C : Normal en charge
   - 🔴 > 75°C : Attention !

3. Regardez l'historique des 24h :
   - Est-ce que ça monte progressivement ? (problème de refroidissement)
   - Ou des pics soudains ? (charges de travail)

### Scénario 3 : "Quand dois-je nettoyer mon disque ?"

**Avec Prometheus + Grafana :**

Dans Prometheus, tapez cette requête :
```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 7*24*3600)
```

Cela vous dit : "Dans 7 jours, combien d'espace restera-t-il ?"

Si le résultat est proche de 0 → Nettoyez maintenant !

---

## Exemples concrets

### Exemple 1 : Créer une alerte "CPU élevé"

**Objectif :** Être alerté si le CPU dépasse 80% pendant 5 minutes.

**Dans Prometheus** (requête PromQL) :
```promql
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
```

**Dans Grafana** :
1. Créez un panel avec cette requête
2. Ajoutez une règle d'alerte
3. Configurez la notification (email, Slack, etc.)

**Résultat :** Vous recevez un email automatiquement !

### Exemple 2 : Comparer "avant" et "après" une optimisation

**Situation :** Vous avez optimisé votre code pour utiliser moins de RAM.

**Avec Grafana :**
1. Notez la date/heure de votre changement
2. Regardez le graphique de mémoire
3. Ajoutez une annotation verticale à cette heure
4. Comparez visuellement avant/après

```
RAM 8GB ┤──────────╮              <- Avant (7GB utilisés)
    6GB ┤          │
    4GB ┤          ╰─────────     <- Après (4GB utilisés)
    2GB ┤                    ▼
        └──────────┬──────────
               Optimisation
```

### Exemple 3 : Trouver pourquoi le réseau est lent

**Dans Grafana**, panel réseau :

```promql
# Trafic entrant (Mbps)
rate(node_network_receive_bytes_total[5m]) * 8 / 1000000

# Trafic sortant (Mbps)
rate(node_network_transmit_bytes_total[5m]) * 8 / 1000000
```

Si vous voyez un pic à 100 Mbps en même temps qu'une lenteur → C'est la bande passante saturée !

---

## FAQ

### Q1 : Ai-je besoin de Prometheus ET Grafana ?

**Oui, ils sont complémentaires :**
- **Prometheus seul** : Vous avez les données mais pas de visualisation pratique
- **Grafana seul** : Vous pouvez faire des dashboards mais sans données
- **Les deux ensemble** : Solution complète ! 🎯

### Q2 : Puis-je voir mes métriques en direct ?

**Oui !** Dans Grafana :
- Cliquez sur l'icône "Refresh" en haut à droite
- Sélectionnez "5s" ou "10s"
- Le dashboard se rafraîchit automatiquement

### Q3 : Combien de temps sont conservées les données ?

**Par défaut : 15 jours**

Après 15 jours, les anciennes données sont automatiquement supprimées pour économiser l'espace disque.

Vous pouvez changer cette durée dans la configuration Prometheus.

### Q4 : Quelle est la différence entre un "panel" et un "dashboard" ?

**Dashboard** = La page entière (comme un tableau de bord de voiture)  
**Panel** = Un graphique individuel (comme un compteur de vitesse)

Un dashboard contient plusieurs panels.

### Q5 : Puis-je surveiller plusieurs serveurs ?

**Oui !** Il suffit d'installer Node Exporter sur chaque serveur et d'ajouter leurs adresses dans `prometheus.yml` :

```yaml
scrape_configs:
  - job_name: 'serveur1'
    static_configs:
      - targets: ['192.168.1.100:9100']
  
  - job_name: 'serveur2'
    static_configs:
      - targets: ['192.168.1.101:9100']
```

### Q6 : Est-ce que ça consomme beaucoup de ressources ?

**Non, c'est très léger :**
- Prometheus : ~50-100 MB RAM, <1% CPU
- Node Exporter : ~10 MB RAM, <0.5% CPU
- Grafana : ~100 MB RAM, <1% CPU
- GPU Exporter : ~10 MB RAM, <0.5% CPU

**Total : ~200 MB RAM et 2-3% CPU maximum**

### Q7 : Puis-je accéder à Grafana depuis mon téléphone ?

**Oui !** Grafana est responsive. Ouvrez simplement `http://192.168.1.108:3000` dans le navigateur de votre téléphone (sur le même réseau WiFi).

### Q8 : Que se passe-t-il si Prometheus tombe en panne ?

- ❌ Vous perdez la collecte pendant la panne
- ✅ Les données déjà collectées sont conservées
- ✅ La collecte reprend automatiquement au redémarrage

**Bonne pratique :** Activez le démarrage automatique avec `systemctl enable prometheus`

### Q9 : Puis-je exporter mes données ?

**Oui, plusieurs méthodes :**

1. **Via API Prometheus** :
```bash
curl 'http://localhost:9090/api/v1/query?query=up'
```

2. **Exporter un dashboard Grafana** :
Settings → JSON Model → Copier

3. **Sauvegarder la base de données** :
```bash
sudo tar -czf prometheus-backup.tar.gz /var/lib/prometheus
```

### Q10 : C'est compliqué de créer un nouveau panel dans Grafana ?

**Non, c'est simple !**

1. Cliquez sur "+ Add" → "Visualization"
2. Entrez votre requête PromQL (ex: `nvidia_gpu_temperature_celsius`)
3. Choisissez le type (Gauge, Time series, etc.)
4. Cliquez sur "Apply"

**Temps estimé : 30 secondes** ⏱️

---

## Résumé en une image

```
┌─────────────────────────────────────────────────────────────┐
│                    VOTRE SYSTÈME HP Z800                    │
│  (CPU, RAM, Disques, GPU fonctionnent normalement)         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Les métriques sont exposées
                     ↓
┌─────────────────────────────────────────────────────────────┐
│               EXPORTERS (Collecteurs)                       │
│  ┌──────────────────┐        ┌──────────────────┐          │
│  │  Node Exporter   │        │  GPU Exporter    │          │
│  │  Métriques       │        │  Métriques       │          │
│  │  système         │        │  GPU             │          │
│  └──────────────────┘        └──────────────────┘          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Prometheus collecte toutes les 15s
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                   PROMETHEUS                                │
│  📊 Stocke toutes les métriques avec leur timestamp         │
│  🔍 Permet de les interroger (PromQL)                       │
│  🔔 Génère des alertes                                      │
│  💾 Garde 15 jours d'historique                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Grafana lit les données
                     ↓
┌─────────────────────────────────────────────────────────────┐
│                    GRAFANA                                  │
│  🎨 Crée de beaux graphiques                                │
│  📈 Dashboards interactifs                                  │
│  👁️ Visualisation en temps réel                            │
│  📱 Accessible depuis n'importe quel navigateur             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Vous consultez depuis Windows
                     ↓
┌─────────────────────────────────────────────────────────────┐
│              VOUS (sur Windows 11)                          │
│  🌐 Ouvrez http://192.168.1.108:3000                        │
│  👀 Regardez vos métriques                                  │
│  🎯 Identifiez les problèmes                                │
│  ✅ Gardez votre système en bonne santé                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Pour aller plus loin

### Documentation recommandée

1. **[PromQL Cheatsheet](PROMQL_CHEATSHEET.md)** - Toutes les requêtes utiles
2. **[Guide d'utilisation](USAGE.md)** - Utilisation détaillée de chaque interface
3. **[Configuration](CONFIGURATION.md)** - Détails techniques et troubleshooting

### Tutoriels pratiques

- [Créer votre premier dashboard](USAGE.md#créer-un-nouveau-dashboard)
- [Configurer des alertes](USAGE.md#configuration-alertes)
- [Importer des dashboards](USAGE.md#importer-dashboards)

### Ressources externes

- [Prometheus Documentation officielle](https://prometheus.io/docs/)
- [Grafana Tutorials](https://grafana.com/tutorials/)
- [PromQL Examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)

---

## En conclusion

**Prometheus** = Le cerveau qui se souvient de tout  
**Grafana** = Les yeux qui vous montrent tout  
**Ensemble** = Un système de monitoring complet et puissant ! 💪

**Vous êtes maintenant prêt à surveiller votre HP Z800 comme un pro !** 🚀
