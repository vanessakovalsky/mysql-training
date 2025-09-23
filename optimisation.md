# TP : Optimisation des paramètres du serveur MySQL
**Durée estimée : 35-45 minutes**

### Objectifs
- Comprendre les paramètres clés de performance MySQL
- Analyser la configuration actuelle
- Optimiser les paramètres selon la charge de travail

### Prérequis
- Accès administrateur MySQL
- Connaissance de la charge de travail applicative
- Outils de monitoring (optionnel : mysqltuner, pt-query-digest)

### Étape 1 : Analyse de la configuration actuelle (10 minutes)

1. **État des variables système importantes :**
```sql
-- Variables de cache et buffer
SELECT 
    VARIABLE_NAME, 
    VARIABLE_VALUE 
FROM performance_schema.global_variables 
WHERE VARIABLE_NAME IN (
    'innodb_buffer_pool_size',
    'key_buffer_size',
    'query_cache_size',
    'sort_buffer_size',
    'read_buffer_size',
    'join_buffer_size',
    'thread_cache_size',
    'table_open_cache',
    'max_connections'
);
```

2. **Informations sur la mémoire système :**
```sql
-- Utilisation actuelle de la mémoire
SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS 'DB Size in MB'
FROM information_schema.tables;

-- Statistiques InnoDB
SELECT 
    VARIABLE_NAME, 
    VARIABLE_VALUE 
FROM performance_schema.global_status 
WHERE VARIABLE_NAME LIKE 'Innodb_%'
AND VARIABLE_NAME IN (
    'Innodb_buffer_pool_read_requests',
    'Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_pages_total',
    'Innodb_buffer_pool_pages_free'
);
```

3. **Installation et utilisation de MySQLTuner (optionnel) :**
```bash
# Téléchargement
wget http://mysqltuner.pl/ -O mysqltuner.pl
chmod +x mysqltuner.pl

# Exécution
perl mysqltuner.pl
```

### Étape 2 : Optimisation du Buffer Pool InnoDB (10 minutes)

1. **Calcul de la taille optimale du buffer pool :**
```sql
-- Taille actuelle
SELECT @@innodb_buffer_pool_size/1024/1024 as 'Buffer Pool Size MB';

-- Taille de données InnoDB
SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS 'InnoDB Size MB'
FROM information_schema.tables 
WHERE engine = 'InnoDB';
```

2. **Configuration recommandée (70-80% de la RAM pour serveur dédié) :**
```ini
[mysqld]
# Pour un serveur avec 4GB RAM
innodb_buffer_pool_size = 3G

# Optimisations associées
innodb_buffer_pool_instances = 4
innodb_log_file_size = 256M
innodb_log_buffer_size = 16M
innodb_flush_log_at_trx_commit = 1
```

3. **Test de performance avant/après :**
```sql
-- Création d'une base de test
CREATE DATABASE perf_test;
USE perf_test;

CREATE TABLE benchmark (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data TEXT,
    number_col INT,
    date_col TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_number (number_col),
    INDEX idx_date (date_col)
);

-- Insertion de données test
INSERT INTO benchmark (data, number_col) 
SELECT 
    CONCAT('Performance test data ', n),
    FLOOR(RAND() * 10000)
FROM (
    SELECT a.N + b.N * 10 + c.N * 100 + d.N * 1000 + 1 n
    FROM 
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b,
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) c,
    (SELECT 0 as N UNION SELECT 1 UNION SELECT 2) d
) numbers;
```

### Étape 3 : Optimisation des paramètres de cache et connexions (10 minutes)

1. **Analyse du cache de tables :**
```sql
-- Statistiques actuelles
SELECT 
    VARIABLE_NAME, 
    VARIABLE_VALUE 
FROM performance_schema.global_status 
WHERE VARIABLE_NAME IN (
    'Open_tables',
    'Opened_tables',
    'Table_open_cache_hits',
    'Table_open_cache_misses'
);

-- Calcul du hit ratio
SELECT 
    ROUND(
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Table_open_cache_hits') / 
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Table_open_cache_hits') + 
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Table_open_cache_misses') * 100, 2
    ) AS 'Table Cache Hit Ratio %';
```

2. **Optimisation des connexions :**
```sql
-- Analyse des connexions
SELECT 
    VARIABLE_NAME, 
    VARIABLE_VALUE 
FROM performance_schema.global_status 
WHERE VARIABLE_NAME IN (
    'Threads_connected',
    'Threads_running',
    'Threads_created',
    'Connection_errors_max_connections'
);

SHOW PROCESSLIST;
```

3. **Configuration optimisée :**
```ini
[mysqld]
# Cache de tables
table_open_cache = 4000
table_definition_cache = 2000

# Gestion des connexions
max_connections = 200
thread_cache_size = 50
max_connect_errors = 100000

# Buffers de tri et jointure
sort_buffer_size = 2M
join_buffer_size = 2M
read_buffer_size = 1M
read_rnd_buffer_size = 2M
```

### Étape 4 : Test de performance et monitoring (10 minutes)

1. **Benchmark simple avec requêtes :**
```sql
-- Test de performance SELECT
SET @start_time = NOW(6);

SELECT COUNT(*), AVG(number_col), MAX(date_col) 
FROM perf_test.benchmark 
WHERE number_col BETWEEN 1000 AND 5000;

SET @end_time = NOW(6);
SELECT TIMESTAMPDIFF(MICROSECOND, @start_time, @end_time) as 'Query Time (microseconds)';
```

2. **Monitoring des performances en temps réel :**
```sql
-- Performance Schema : top requêtes
SELECT 
    TRUNCATE(TIMER_WAIT/1000000000000,6) as exec_time,
    SQL_TEXT
FROM performance_schema.events_statements_history_long 
ORDER BY TIMER_WAIT DESC 
LIMIT 5;

-- Buffer pool hit ratio
SELECT 
    ROUND(
        (1 - (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') / 
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')) * 100, 2
    ) AS 'InnoDB Buffer Pool Hit Ratio %';
```

3. **Script de monitoring personnalisé :**
```sql
-- Création d'une vue de monitoring
CREATE OR REPLACE VIEW performance_monitoring AS
SELECT 
    'Buffer Pool Hit Ratio' as metric,
    CONCAT(
        ROUND((1 - gs1.VARIABLE_VALUE / gs2.VARIABLE_VALUE) * 100, 2), '%'
    ) as value
FROM 
    performance_schema.global_status gs1,
    performance_schema.global_status gs2
WHERE 
    gs1.VARIABLE_NAME = 'Innodb_buffer_pool_reads'
    AND gs2.VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'

UNION ALL

SELECT 
    'Query Cache Hit Ratio' as metric,
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Qcache_hits') / 
            ((SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Qcache_hits') + 
             (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Com_select')) * 100, 2
        ), '%'
    ) as value

UNION ALL

SELECT 
    'Thread Cache Hit Ratio' as metric,
    CONCAT(
        ROUND(
            (1 - (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Threads_created') / 
            (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Connections')) * 100, 2
        ), '%'
    ) as value;

-- Utilisation de la vue
SELECT * FROM performance_monitoring;
```

### Questions/Exercices pratiques (5 minutes)

1. **Calculez le ratio de hit du buffer pool InnoDB actuel**
2. **Déterminez si votre serveur a besoin de plus de connexions simultanées**
3. **Identifiez le paramètre le plus critique à ajuster selon votre charge de travail**
4. **Établissez un plan de monitoring régulier des performances**

### Configuration finale recommandée
```ini
[mysqld]
# Mémoire et cache
innodb_buffer_pool_size = 3G
innodb_buffer_pool_instances = 4
key_buffer_size = 256M
query_cache_size = 64M
table_open_cache = 4000

# Logs et performances
innodb_log_file_size = 256M
innodb_log_buffer_size = 16M
slow_query_log = 1
long_query_time = 2

# Connexions
max_connections = 200
thread_cache_size = 50

# Buffers
sort_buffer_size = 2M
join_buffer_size = 2M
read_buffer_size = 1M
```
