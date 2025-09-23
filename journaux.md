# TP : Activation et Analyse des journaux MySQL
**Durée estimée : 30-45 minutes**

### Objectifs
- Comprendre les différents types de journaux MySQL
- Activer et configurer les journaux
- Analyser les informations des journaux pour le monitoring et le dépannage

### Prérequis
- Accès root ou privilèges administrateur sur MySQL
- MySQL Server installé et fonctionnel
- Accès au système de fichiers du serveur

### Étape 1 : Vérification de l'état actuel des journaux (5 minutes)

1. **Connexion à MySQL en tant qu'administrateur :**
```sql
mysql -u root -p
```

2. **Vérification des journaux actuellement activés :**
```sql
SHOW VARIABLES LIKE '%log%';
```

3. **Vérification spécifique des journaux principaux :**
```sql
-- Journal des erreurs
SHOW VARIABLES LIKE 'log_error%';

-- Journal général
SHOW VARIABLES LIKE 'general_log%';

-- Journal des requêtes lentes
SHOW VARIABLES LIKE 'slow_query_log%';

-- Journal binaire
SHOW VARIABLES LIKE 'log_bin%';
```

### Étape 2 : Configuration du journal général (10 minutes)

1. **Activation du journal général (session courante) :**
```sql
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/mysql-general.log';
```

2. **Configuration permanente dans my.cnf :**
```ini
[mysqld]
general_log = 1
general_log_file = /var/log/mysql/mysql-general.log
```

3. **Test et vérification :**
```sql
-- Exécuter quelques requêtes de test
SELECT NOW();
USE mysql;
SELECT COUNT(*) FROM user;

-- Vérifier que le journal enregistre
SHOW VARIABLES LIKE 'general_log%';
```

4. **Consultation du journal :**
```bash
# En dehors de MySQL
tail -f /var/log/mysql/mysql-general.log
```

### Étape 3 : Configuration du journal des requêtes lentes (10 minutes)

1. **Activation et configuration :**
```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/mysql-slow.log';
SET GLOBAL long_query_time = 2; -- 2 secondes
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

2. **Configuration permanente dans my.cnf :**
```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
log_queries_not_using_indexes = 1
```

3. **Génération de requêtes lentes pour test :**
```sql
-- Créer une table de test
CREATE DATABASE IF NOT EXISTS test_logs;
USE test_logs;

CREATE TABLE test_table (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer des données
INSERT INTO test_table (data) 
SELECT CONCAT('Test data ', n) 
FROM (
    SELECT a.N + b.N * 10 + c.N * 100 + 1 n
    FROM 
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b,
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) c
) numbers;

-- Requête lente intentionnelle (sans index)
SELECT * FROM test_table WHERE data LIKE '%500%';
SELECT SLEEP(3);
```

4. **Analyse du journal des requêtes lentes :**
```bash
# Utilisation de mysqldumpslow
mysqldumpslow -s t /var/log/mysql/mysql-slow.log
mysqldumpslow -s c /var/log/mysql/mysql-slow.log
```

### Étape 4 : Gestion du journal binaire (10 minutes)

1. **Vérification du journal binaire :**
```sql
SHOW VARIABLES LIKE 'log_bin%';
SHOW MASTER STATUS;
SHOW BINARY LOGS;
```

2. **Si le binlog n'est pas activé, configuration dans my.cnf :**
```ini
[mysqld]
log_bin = /var/log/mysql/mysql-bin
server_id = 1
binlog_format = ROW
expire_logs_days = 7
```

3. **Analyse du contenu du binlog :**
```bash
# Affichage du contenu binaire
mysqlbinlog /var/log/mysql/mysql-bin.000001

# Avec filtrage par base de données
mysqlbinlog --database=test_logs /var/log/mysql/mysql-bin.000001
```

4. **Gestion de la rotation des binlogs :**
```sql
-- Forcer une rotation
FLUSH BINARY LOGS;

-- Purger les anciens logs
PURGE BINARY LOGS BEFORE DATE(NOW() - INTERVAL 3 DAY);
```

### Questions/Exercices pratiques (5-10 minutes)

1. **Identifiez dans le journal général :**
   - Les connexions d'utilisateurs
   - Les requêtes DDL exécutées
   - Les déconnexions

2. **Analysez le journal des requêtes lentes :**
   - Quelle est la requête la plus lente ?
   - Combien de requêtes n'utilisent pas d'index ?

3. **Calculez l'espace disque utilisé par tous les journaux**



