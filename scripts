# Créer le script
sudo tee /usr/local/bin/prometheus-reset.sh > /dev/null <<'EOF'
#!/bin/bash
echo "🛑 Arrêt de Prometheus..."
sudo systemctl stop prometheus

echo "🗑️  Suppression des données corrompues..."
sudo rm -rf /mnt/data/prometheus/data/*

echo "📁 Recréation de la structure..."
sudo mkdir -p /mnt/data/prometheus/data
sudo chown -R prometheus:prometheus /mnt/data/prometheus

echo "🚀 Redémarrage de Prometheus..."
sudo systemctl start prometheus

echo "⏳ Attente de 30 secondes..."
sleep 30

echo "✅ Test de la collecte..."
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result | length'

echo ""
echo "✅ Prometheus réinitialisé avec succès !"
echo "   Ouvrez Grafana et changez la période pour 'Last 5 minutes'"
EOF

# Rendre exécutable
sudo chmod +x /usr/local/bin/prometheus-reset.sh
