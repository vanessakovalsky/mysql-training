# TP : Configuration de Réplication Simple (45 minutes)

## 🎯 Objectifs
- Configurer une réplication maître-esclave
- Tester la synchronisation des données
- Monitorer l'état de la réplication

## 📋 **Partie A : Préparation (simulation avec un seul serveur)** (15 min)

**Note :** Ce TP simule la réplication sur un seul serveur avec deux instances MySQL. En production, utilisez des serveurs séparés.

**Étape 1 : Créer une seconde instance MySQL (simulation)**
```bash
# Créer les répertoires pour l'instance esclave
sudo mkdir -p /var/log/mysql-slave
sudo mkdir -p /var/run/mysqld-slave

# Copier la configuration pour l'esclave
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld-slave.cnf

# Modifier la configuration de l'esclave
sudo tee /etc/mysql/mysql.conf.d/mysqld-slave.cnf > /dev/null << 'EOF'
[mysqld]
# Configuration pour l'instance esclave (simulation)
port = 3307
server-id = 2
socket = /var/run/mysqld-slave/mysqld.sock
pid-file = /var/run/mysqld-slave/mysqld.pid
datadir = /var/lib/mysql-slave
tmpdir = /tmp

# Journal de relai
relay-log = /var/log/mysql-slave/mysql-relay
relay-log-index = /var/log/mysql-slave/mysql-relay.index

# Lecture seule
read_only = 1
super_read_only = 1

# Binlog pour l'esclave (optionnel)
log-bin = /var/log/mysql-slave/mysql-bin
binlog_format = ROW

# Sécurité de la réplication
relay_log_recovery = 1

# Logs
log-error = /var/log/mysql-slave/error.log

[client]
port = 3307
socket = /var/run/mysqld-slave/mysqld.sock
EOF

# Ajuster les permissions
sudo chown -R mysql:mysql /var/lib/mysql-slave
sudo chown -R mysql:mysql /var/log/mysql-slave
sudo chown -R mysql:mysql /var/run/mysqld-slave
```

**Étape 2 : Initialiser l'instance esclave**
```bash
# Initialiser l'instance esclave
mysqld --defaults-file=/etc/mysql/mysql.conf.d/mysqld-slave.cnf --initialize-insecure --user=mysql

sudo mkdir -p /var/lib/mysql-slave

# Démarrer l'instance esclave
mysqld --defaults-file=/etc/mysql/mysql.conf.d/mysqld-slave.cnf --user=mysql --daemonize

# Vérifier que l'esclave est démarré
ps aux | grep mysqld

# Tester la connexion à l'esclave
mysql -u root -h 127.0.0.1 -P 3307 -e "SELECT 'Esclave connecté' as status;"
```

## 📋 **Partie B : Configuration du maître** (10 min)

**Étape 1 : Préparer le serveur maître**
```sql
-- Se connecter au serveur maître (port 3306)
mysql -u formation -p

-- Créer l'utilisateur de réplication
CREATE USER 'replica_user'@'%' IDENTIFIED BY 'ReplicaPass2024!';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;

-- Vérifier la configuration du binlog
SHOW VARIABLES LIKE 'server_id';
SHOW VARIABLES LIKE 'log_bin';

-- Verrouiller les tables pour obtenir une position cohérente
FLUSH TABLES WITH READ LOCK;

-- Noter la position actuelle du maître
SHOW MASTER STATUS;

-- ⚠️ IMPORTANT: Noter le File et Position affichés
-- Exemple: File: mysql-bin.000001, Position: 1234

EXIT;
```

**Étape 2 : Créer une sauvegarde pour initialiser l'esclave**
```bash
# Créer la sauvegarde du maître (avec la position)
mysqldump -u formation -p \
    --single-transaction \
    --master-data=2 \
    --triggers \
    --routines \
    --all-databases > master_for_slave.sql

# Déverrouiller les tables du maître
mysql -u formation -p -e "UNLOCK TABLES;"

# Examiner les informations de réplication dans la sauvegarde
grep "CHANGE MASTER TO" master_for_slave.sql
```

## 📋 **Partie C : Configuration de l'esclave** (10 min)

**Étape 1 : Initialiser l'esclave avec les données du maître**
```bash
# Restaurer les données du maître sur l'esclave
mysql -u root -h 127.0.0.1 -P 3307 < master_for_slave.sql

# Vérifier que les données sont présentes
mysql -u root -h 127.0.0.1 -P 3307 -e "
SHOW DATABASES;
USE sauvegarde_test;
SELECT COUNT(*) as nb_commandes FROM commandes;
"
```

**Étape 2 : Configurer la réplication**
```sql
-- Se connecter à l'esclave
mysql -u root -h 127.0.0.1 -P 3307

-- Configurer la connexion au maître
CHANGE MASTER TO
    MASTER_HOST = '127.0.0.1',
    MASTER_PORT = 3306,
    MASTER_USER = 'replica_user',
    MASTER_PASSWORD = 'ReplicaPass2024!',
    MASTER_LOG_FILE = 'mysql-bin.000001',  -- Remplacer par la valeur notée
    MASTER_LOG_POS = 1234;                 -- Remplacer par la position notée

-- Démarrer la réplication
START SLAVE;

-- Vérifier l'état de la réplication
SHOW SLAVE STATUS\G;

-- Points à vérifier :
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0 ou petit nombre
-- Last_Error: (doit être vide)

EXIT;
```

## 📋 **Partie D : Test de la réplication** (10 min)

**Étape 1 : Tester la synchronisation des données**
```sql
-- Sur le MAÎTRE (port 3306)
mysql -u formation -p

USE sauvegarde_test;

-- Ajouter des données sur le maître
INSERT INTO commandes (numero, client_nom, montant) VALUES 
('REPLI-001', 'Test Réplication 1', 123.45),
('REPLI-002', 'Test Réplication 2', 678.90);

-- Mettre à jour des données existantes
UPDATE commandes SET montant = 999.99 WHERE numero = 'REPLI-001';

-- Créer une nouvelle table
CREATE TABLE test_replication (
    id INT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_replication (message) VALUES 
('Message depuis le maître'),
('Deuxième message de test');

EXIT;
```

```sql
-- Sur l'ESCLAVE (port 3307) - vérifier la synchronisation
mysql -u root -h 127.0.0.1 -P 3307

USE sauvegarde_test;

-- Vérifier que les nouvelles données sont arrivées
SELECT * FROM commandes WHERE numero LIKE 'REPLI%';

-- Vérifier la nouvelle table
SELECT * FROM test_replication;

-- Vérifier l'état de la réplication
SHOW SLAVE STATUS\G;

EXIT;
```

**Étape 2 : Monitoring de la réplication**
```bash
# Script de monitoring de réplication
cat > monitor_replication.sh << 'EOF'
#!/bin/bash

echo "=== Monitoring Réplication MySQL ==="
echo "Date: $(date)"

# État du maître
echo -e "\n--- MAÎTRE (Port 3306) ---"
mysql -u formation -p -e "SHOW MASTER STATUS;"

# État de l'esclave
echo -e "\n--- ESCLAVE (Port 3307) ---"
mysql -u root -h 127.0.0.1 -P 3307 -e "
SHOW SLAVE STATUS\G;" | grep -E "(Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Last_Error):"

# Test rapide de synchronisation
echo -e "\n--- Test de Synchronisation ---"
TIMESTAMP=$(date +%s)
mysql -u formation -p -e "
USE sauvegarde_test;
INSERT INTO test_replication (message) VALUES ('Test sync $TIMESTAMP');
"

sleep 2

SYNC_TEST=$(mysql -u root -h 127.0.0.1 -P 3307 -e "
USE sauvegarde_test;
SELECT COUNT(*) FROM test_replication WHERE message LIKE '%$TIMESTAMP%';
" 2>/dev/null | tail -1)

if [ "$SYNC_TEST" = "1" ]; then
    echo "✓ Synchronisation OK"
else
    echo "✗ Problème de synchronisation"
fi

EOF

chmod +x monitor_replication.sh

# Exécuter le monitoring
./monitor_replication.sh
```

---
