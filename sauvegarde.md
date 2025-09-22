# TP Sauvegardes avec MYSQLDUMP/MYSQLPUMP (60 minutes)

## 🎯 Objectifs
- Maîtriser les différents types de sauvegardes logiques
- Comparer mysqldump et mysqlpump
- Créer des scripts de sauvegarde automatisés
- Tester les restaurations

## 📋 **Partie A : Préparation de l'environnement** (10 min)

**Étape 1 : Créer un environnement de test avec données**
```sql
-- Se connecter à MySQL
mysql -u formation -p

-- Créer une base de données de test
CREATE DATABASE sauvegarde_test;
USE sauvegarde_test;

-- Créer des tables avec données significatives
CREATE TABLE commandes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    numero VARCHAR(20) NOT NULL,
    client_nom VARCHAR(100),
    montant DECIMAL(10,2),
    date_commande TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE produits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(8,2),
    stock INT DEFAULT 0,
    description TEXT
) ENGINE=InnoDB;

-- Fonction pour générer des données
DELIMITER //
CREATE PROCEDURE generer_donnees_test(IN nb_commandes INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= nb_commandes DO
        INSERT INTO commandes (numero, client_nom, montant) VALUES
        (CONCAT('CMD-2024-', LPAD(i, 6, '0')), 
         CONCAT('Client-', i), 
         ROUND(RAND() * 1000 + 50, 2));
        SET i = i + 1;
    END WHILE;
END //

CREATE PROCEDURE generer_produits_test(IN nb_produits INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= nb_produits DO
        INSERT INTO produits (nom, prix, stock, description) VALUES
        (CONCAT('Produit-', i), 
         ROUND(RAND() * 500 + 10, 2),
         FLOOR(RAND() * 100),
         CONCAT('Description détaillée du produit ', i));
        SET i = i + 1;
    END WHILE;
END //
DELIMITER ;

-- Générer les données de test
CALL generer_donnees_test(1000);
CALL generer_produits_test(200);

-- Vérifier les données créées
SELECT COUNT(*) as nb_commandes FROM commandes;
SELECT COUNT(*) as nb_produits FROM produits;
SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as taille_mb
FROM information_schema.tables 
WHERE table_schema = 'sauvegarde_test';
```

**Étape 2 : Créer le répertoire de sauvegarde**
```bash
# Créer le répertoire de sauvegarde
sudo mkdir -p /backup/mysql
sudo chown $(whoami):$(whoami) /backup/mysql
cd /backup/mysql

# Vérifier l'espace disque disponible
df -h /backup
```

## 📋 **Partie B : Sauvegardes avec mysqldump** (25 min)

**Étape 1 : Sauvegarde basique**
```bash
# Sauvegarde complète d'une base
mysqldump -u formation -p sauvegarde_test > backup_basic_$(date +%Y%m%d_%H%M%S).sql

# Vérifier la taille du fichier créé
ls -lh backup_basic_*.sql

# Examiner le début du fichier
head -20 backup_basic_*.sql
```

**Étape 2 : Sauvegarde avec options avancées**
```bash
# Sauvegarde avec transaction consistante (pour InnoDB)
mysqldump -u formation -p \
    --single-transaction \
    --triggers \
    --routines \
    --events \
    --create-options \
    --extended-insert \
    sauvegarde_test > backup_advanced_$(date +%Y%m%d_%H%M%S).sql

# Sauvegarde avec compression
mysqldump -u formation -p \
    --single-transaction \
    --triggers \
    --routines \
    sauvegarde_test | gzip > backup_compressed_$(date +%Y%m%d_%H%M%S).sql.gz

# Comparer les tailles
ls -lh backup_*
```

**Étape 3 : Sauvegardes sélectives**
```bash
# Structure seule (sans données)
mysqldump -u formation -p \
    --no-data \
    --routines \
    --triggers \
    sauvegarde_test > structure_only_$(date +%Y%m%d_%H%M%S).sql

# Données seules (sans structure)
mysqldump -u formation -p \
    --no-create-info \
    --complete-insert \
    sauvegarde_test > data_only_$(date +%Y%m%d_%H%M%S).sql

# Une table spécifique seulement
mysqldump -u formation -p \
    --single-transaction \
    sauvegarde_test commandes > table_commandes_$(date +%Y%m%d_%H%M%S).sql

# Certaines lignes seulement (avec WHERE)
mysqldump -u formation -p \
    --single-transaction \
    --where="date_commande >= '2024-01-01'" \
    sauvegarde_test commandes > commandes_2024_$(date +%Y%m%d_%H%M%S).sql
```

**Étape 4 : Script de sauvegarde automatisé**
```bash
# Créer un script de sauvegarde complet
cat > backup_script.sh << 'EOF'
#!/bin/bash

# Configuration
DB_USER="formation"
DB_PASS_FILE="/home/$(whoami)/.mysql_password"  # Créer ce fichier avec le mot de passe
BACKUP_DIR="/backup/mysql"
DATABASE="sauvegarde_test"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# Créer le répertoire si nécessaire
mkdir -p "$BACKUP_DIR"

# Fonction de log
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$BACKUP_DIR/backup.log"
}

# Vérifications préalables
if [ ! -f "$DB_PASS_FILE" ]; then
    log_message "ERREUR: Fichier de mot de passe non trouvé: $DB_PASS_FILE"
    exit 1
fi

DB_PASS=$(cat "$DB_PASS_FILE")

# Test de connexion
if ! mysql -u "$DB_USER" -p"$DB_PASS" -e "SELECT 1;" >/dev/null 2>&1; then
    log_message "ERREUR: Impossible de se connecter à MySQL"
    exit 1
fi

log_message "Début de la sauvegarde de $DATABASE"

# Sauvegarde complète avec compression
BACKUP_FILE="$BACKUP_DIR/${DATABASE}_full_${DATE}.sql.gz"
if mysqldump -u "$DB_USER" -p"$DB_PASS" \
    --single-transaction \
    --triggers \
    --routines \
    --events \
    --create-options \
    "$DATABASE" | gzip > "$BACKUP_FILE"; then
    
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log_message "Sauvegarde réussie: $BACKUP_FILE ($BACKUP_SIZE)"
else
    log_message "ERREUR: Échec de la sauvegarde"
    exit 1
fi

# Nettoyage des anciennes sauvegardes
find "$BACKUP_DIR" -name "${DATABASE}_full_*.sql.gz" -mtime +$RETENTION_DAYS -delete
log_message "Nettoyage des sauvegardes > $RETENTION_DAYS jours"

# Statistiques finales
NB_BACKUPS=$(find "$BACKUP_DIR" -name "${DATABASE}_full_*.sql.gz" | wc -l)
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log_message "Fin de sauvegarde. $NB_BACKUPS fichiers présents, total: $TOTAL_SIZE"

EOF

# Rendre le script exécutable
chmod +x backup_script.sh

# Créer le fichier de mot de passe
echo "FormationTP2024!" > ~/.mysql_password
chmod 600 ~/.mysql_password

# Tester le script
./backup_script.sh
```

## 📋 **Partie C : Sauvegardes avec mysqlpump (MySQL 5.7+)** (15 min)

**Étape 1 : Sauvegarde parallélisée avec mysqlpump**
```bash
# Vérifier la disponibilité de mysqlpump
which mysqlpump
mysqlpump --version

# Sauvegarde parallélisée (4 threads)
mysqlpump -u formation -p \
    --single-transaction \
    --default-parallelism=4 \
    --compress-output=zlib \
    sauvegarde_test > backup_mysqlpump_$(date +%Y%m%d_%H%M%S).sql.zlib

# Sauvegarde avec exclusions spécifiques
mysqlpump -u formation -p \
    --exclude-tables=sauvegarde_test.commandes \
    --default-parallelism=2 \
    sauvegarde_test > backup_exclude_commandes_$(date +%Y%m%d_%H%M%S).sql
```

**Étape 2 : Comparaison de performances**
```bash
# Test de performance mysqldump
time mysqldump -u formation -p \
    --single-transaction \
    sauvegarde_test > /tmp/test_mysqldump.sql

# Test de performance mysqlpump
time mysqlpump -u formation -p \
    --single-transaction \
    --default-parallelism=4 \
    sauvegarde_test > /tmp/test_mysqlpump.sql

# Comparer les tailles
ls -lh /tmp/test_mysql*.sql

# Nettoyer les fichiers de test
rm /tmp/test_mysql*.sql
```

## 📋 **Partie D : Tests de restauration** (10 min)

**Étape 1 : Restauration basique**
```sql
-- Créer une base de test pour la restauration
mysql -u formation -p

CREATE DATABASE restauration_test;
EXIT;
```

```bash
# Restaurer depuis une sauvegarde
mysql -u formation -p restauration_test < backup_basic_*.sql

# Vérifier la restauration
mysql -u formation -p -e "
USE restauration_test;
SELECT COUNT(*) as nb_commandes FROM commandes;
SELECT COUNT(*) as nb_produits FROM produits;
"
```

**Étape 2 : Restauration avec décompression**
```bash
# Restaurer depuis une sauvegarde compressée
gunzip < backup_compressed_*.sql.gz | mysql -u formation -p restauration_test

# Restaurer depuis mysqlpump avec décompression zlib
# (nécessite un outil de décompression zlib)
```

**✅ Point de contrôle :** Vérifier que toutes les données ont été correctement restaurées.

---
