# TP 2 : Point-in-Time Recovery (PITR) (50 minutes)

## 🎯 Objectifs
- Configurer et utiliser le journal binaire
- Effectuer une restauration à un point dans le temps
- Simuler un incident et récupérer les données

## 📋 **Partie A : Configuration du journal binaire** (15 min)

**Étape 1 : Vérifier et configurer le binlog**
```sql
-- Vérifier l'état actuel du binlog
mysql -u formation -p

SHOW VARIABLES LIKE 'log_bin%';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'server_id';

-- Si le binlog n'est pas activé, configurer dans my.cnf
EXIT;
```

```bash
# Sauvegarder la configuration actuelle
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.backup

# Ajouter la configuration binlog (si pas déjà présent)
sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf << EOF

# Configuration binlog pour PITR
server-id = 1
log-bin = /var/log/mysql/mysql-bin
binlog_format = ROW
max_binlog_size = 100M
binlog_expire_logs_seconds = 604800
sync_binlog = 1

EOF

# Redémarrer MySQL pour appliquer la configuration
sudo systemctl restart mysql

# Vérifier que MySQL a bien redémarré
sudo systemctl status mysql
```

**Étape 2 : Vérifier la configuration du binlog**
```sql
mysql -u formation -p

-- Vérifier que le binlog est maintenant actif
SHOW VARIABLES LIKE 'log_bin';
SHOW MASTER STATUS;

-- Voir les fichiers binlog disponibles
SHOW BINARY LOGS;

-- Créer quelques transactions pour peupler le binlog
USE sauvegarde_test;
INSERT INTO commandes (numero, client_nom, montant) VALUES 
('PITR-001', 'Client PITR Test 1', 150.00),
('PITR-002', 'Client PITR Test 2', 275.50);

-- Voir les événements récents dans le binlog
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 10;

EXIT;
```

## 📋 **Partie B : Créer un scénario de récupération** (20 min)

**Étape 1 : Créer une sauvegarde de référence**
```bash
# Créer une sauvegarde complète à 14h00 (simulation)
BACKUP_TIME="2024-$(date +%m-%d)_14:00:00"
echo "=== Sauvegarde de référence à 14:00:00 ==="

mysqldump -u formation -p \
    --single-transaction \
    --flush-logs \
    --master-data=2 \
    --triggers \
    --routines \
    sauvegarde_test > backup_reference_14h00.sql

# Noter la position dans le binlog
grep "CHANGE MASTER TO" backup_reference_14h00.sql
```

**Étape 2 : Simuler une activité normale**
```sql
mysql -u formation -p

USE sauvegarde_test;

-- Simuler des opérations normales à 14:30
INSERT INTO commandes (numero, client_nom, montant) VALUES 
('NORMAL-001', 'Commande Normale 1', 89.99),
('NORMAL-002', 'Commande Normale 2', 156.75),
('NORMAL-003', 'Commande Normale 3', 234.50);

INSERT INTO produits (nom, prix, stock) VALUES 
('Produit Normal 1', 45.00, 25),
('Produit Normal 2', 67.50, 15);

UPDATE produits SET stock = stock - 1 WHERE id = 1;
UPDATE produits SET prix = 47.00 WHERE nom = 'Produit Normal 1';

-- Noter l'heure actuelle (simulation 14:30)
SELECT NOW() as heure_operations_normales;

EXIT;
```

**Étape 3 : Simuler un incident à 14:45**
```sql
mysql -u formation -p

USE sauvegarde_test;

-- Simuler quelques opérations valides avant l'incident
INSERT INTO commandes (numero, client_nom, montant) VALUES 
('AVANT-INCIDENT-001', 'Dernière commande valide', 99.99);

-- NOTER L'HEURE EXACTE AVANT L'INCIDENT
SELECT NOW() as heure_avant_incident;

-- ⚠️ SIMULATION D'INCIDENT : Suppression accidentelle ⚠️
DELETE FROM commandes WHERE montant > 100;  -- Oops, suppression massive!
DROP TABLE produits;  -- Double oops!

-- Constater les dégâts
SELECT COUNT(*) as commandes_restantes FROM commandes;
SHOW TABLES;  -- La table produits a disparu

EXIT;
```

## 📋 **Partie C : Récupération Point-in-Time** (15 min)

**Étape 1 : Analyser les binlogs pour trouver le point de récupération**
```bash
# Lister les fichiers binlog disponibles
mysql -u formation -p -e "SHOW BINARY LOGS;"

# Examiner les événements dans le binlog pour trouver l'incident
mysqlbinlog /var/log/mysql/mysql-bin.000001 | grep -i -A5 -B5 "DELETE FROM commandes"

# Examiner avec une plage temporelle
mysqlbinlog --start-datetime="$(date +%Y-%m-%d) 14:00:00" \
           --stop-datetime="$(date +%Y-%m-%d) 15:00:00" \
           /var/log/mysql/mysql-bin.000001 | head -50

# Trouver la position exacte avant l'incident (on va utiliser l'heure)
RECOVERY_TIME="$(date +%Y-%m-%d) 14:44:00"  # Juste avant l'incident
echo "Point de récupération choisi: $RECOVERY_TIME"
```

**Étape 2 : Effectuer la restauration PITR**
```bash
echo "=== Début de la récupération Point-in-Time ==="

# 1. Supprimer la base corrompue
mysql -u formation -p -e "DROP DATABASE sauvegarde_test;"

# 2. Recréer la base
mysql -u formation -p -e "CREATE DATABASE sauvegarde_test;"

# 3. Restaurer la sauvegarde de référence (14h00)
echo "Restauration de la sauvegarde de référence..."
mysql -u formation -p sauvegarde_test < backup_reference_14h00.sql

# 4. Rejouer les transactions du binlog jusqu'au point de récupération
echo "Rejeu des transactions jusqu'à $RECOVERY_TIME..."
mysqlbinlog --start-datetime="$(date +%Y-%m-%d) 14:00:00" \
           --stop-datetime="$RECOVERY_TIME" \
           /var/log/mysql/mysql-bin.000001 | mysql -u formation -p

echo "=== Récupération terminée ==="
```

**Étape 3 : Vérifier la récupération**
```sql
mysql -u formation -p

USE sauvegarde_test;

-- Vérifier que les données sont récupérées
SELECT COUNT(*) as nb_commandes_recuperees FROM commandes;
SELECT COUNT(*) as nb_produits_recuperes FROM produits;

-- Vérifier que les opérations normales sont présentes
SELECT * FROM commandes WHERE numero LIKE 'NORMAL%';
SELECT * FROM commandes WHERE numero LIKE 'AVANT-INCIDENT%';

-- Vérifier que les données corrompues ne sont PAS là
SELECT * FROM commandes WHERE montant > 200;  -- Devrait être vide ou presque

-- Voir les dernières commandes
SELECT * FROM commandes ORDER BY date_commande DESC LIMIT 10;

EXIT;
```
