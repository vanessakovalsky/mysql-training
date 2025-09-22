## TP  : Configuration et Variables Système (30 minutes)

#### 🎯 Objectifs
- Localiser et modifier les fichiers de configuration
- Comprendre les variables système importantes
- Personnaliser la configuration MySQL

---

#### 📋 **Partie A : Localisation des fichiers de configuration** (10 min)

**Sur Linux :**
```bash
# Trouver le fichier de configuration actuel
mysql --help | grep "Default options" -A 1

# Localiser le fichier principal (généralement)
ls -la /etc/mysql/mysql.conf.d/mysqld.cnf

# Voir le contenu
sudo cat /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Sur Windows :**
- Fichier principal : `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini`
- Ouvrir avec un éditeur de texte (en tant qu'administrateur)

**Étape 1 : Identifier les paramètres actuels**
```sql
-- Se connecter à MySQL
mysql -u formation -p

-- Voir le répertoire de données
SHOW VARIABLES LIKE 'datadir';

-- Voir le fichier de configuration utilisé
SHOW VARIABLES LIKE 'pid_file';

-- Voir les paramètres de mémoire
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

---

#### 📋 **Partie B : Variables système importantes** (15 min)

**Étape 1 : Explorer les variables de configuration**
```sql
-- Variables de performance
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'query_cache_size';

-- Variables de stockage
SHOW VARIABLES LIKE 'innodb_file_per_table';
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';

-- Variables de sécurité
SHOW VARIABLES LIKE 'sql_mode';
SHOW VARIABLES LIKE 'local_infile';
```

**Étape 2 : Modifier des variables en session**
```sql
-- Voir le mode SQL actuel
SELECT @@sql_mode;

-- Modifier le mode SQL pour la session
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER';

-- Vérifier le changement
SELECT @@sql_mode;

-- Tester l'impact (insertion d'une date invalide)
CREATE DATABASE test_config;
USE test_config;

CREATE TABLE test_strict (
    id INT,
    date_test DATE
);

-- Ceci va échouer avec le mode strict
INSERT INTO test_strict VALUES (1, '2024-13-40');
```

**Étape 3 : Variables globales (nécessite privilèges)**
```sql
-- Voir le nombre max de connexions
SELECT @@max_connections;

-- Le modifier (temporairement)
SET GLOBAL max_connections = 250;

-- Vérifier le changement
SELECT @@max_connections;

-- Voir qui est connecté
SHOW PROCESSLIST;
```

---

#### 📋 **Partie C : Modification permanente de la configuration** (5 min)

**Linux - Éditer le fichier de configuration :**
```bash
# Sauvegarder la configuration actuelle
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.backup

# Éditer le fichier
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Ajouter dans la section [mysqld] :**
```ini
# Configuration de formation
max_connections = 250
innodb_buffer_pool_size = 256M

# Logs pour le debug
general_log = 1
general_log_file = /var/log/mysql/mysql.log
```

**Redémarrer MySQL :**
```bash
sudo systemctl restart mysql
```

**Windows - Éditer my.ini :**
```ini
[mysqld]
max_connections=250
innodb_buffer_pool_size=256M
general_log=1
```

**Redémarrer le service MySQL dans les Services Windows.**

**✅ Vérification :**
```sql
mysql -u formation -p
SELECT @@max_connections;
-- Doit afficher 250
```

## Validation TP2 - Configuration
- [ ] Fichier de configuration localisé et modifié
- [ ] Variables système consultées et modifiées
- [ ] Différence entre variables session et globales comprise
- [ ] Redémarrage du service réussi avec nouvelle configuration
