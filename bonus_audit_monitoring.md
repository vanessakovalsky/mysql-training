### Défi 4 : Système d'Audit et Monitoring Complet

#### 🎯 Objectif
Créer un système complet d'audit et de monitoring des performances.

```sql
-- Table de monitoring des performances
CREATE TABLE performance_monitoring (
    id INT PRIMARY KEY AUTO_INCREMENT,
    query_type ENUM('SELECT', 'INSERT', 'UPDATE', 'DELETE', 'PROCEDURE'),
    query_hash VARCHAR(64), -- Hash MD5 de la requête
    execution_time_ms INT,
    rows_examined INT,
    rows_sent INT,
    table_name VARCHAR(100),
    user_name VARCHAR(100),
    timestamp_execution TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_execution_time (execution_time_ms),
    INDEX idx_timestamp (timestamp_execution),
    INDEX idx_user (user_name)
) ENGINE=InnoDB;

-- Procédure pour logger les performances
DELIMITER //

CREATE PROCEDURE log_performance(
    IN p_query_type ENUM('SELECT', 'INSERT', 'UPDATE', 'DELETE', 'PROCEDURE'),
    IN p_query_text TEXT,
    IN p_execution_time_ms INT,
    IN p_rows_examined INT,
    IN p_rows_sent INT,
    IN p_table_name VARCHAR(100)
)
BEGIN
    DECLARE v_query_hash VARCHAR(64);
    
    -- Créer un hash de la requête pour éviter de stocker le texte complet
    SET v_query_hash = MD5(p_query_text);
    
    -- Insérer les données de performance
    INSERT INTO performance_monitoring 
    (query_type, query_hash, execution_time_ms, rows_examined, rows_sent, table_name, user_name)
    VALUES 
    (p_query_type, v_query_hash, p_execution_time_ms, p_rows_examined, p_rows_sent, p_table_name, USER());
    
END //

DELIMITER ;

-- Vue pour les statistiques de performance
CREATE VIEW stats_performance AS
SELECT 
    query_type,
    table_name,
    COUNT(*) as nb_executions,
    AVG(execution_time_ms) as temps_moyen_ms,
    MIN(execution_time_ms) as temps_min_ms,
    MAX(execution_time_ms) as temps_max_ms,
    AVG(rows_examined) as lignes_examinees_moy,
    AVG(rows_sent) as lignes_retournees_moy,
    DATE(timestamp_execution) as date_exec
FROM performance_monitoring
GROUP BY query_type, table_name, DATE(timestamp_execution)
ORDER BY temps_moyen_ms DESC;

-- Procédure de rapport de performance
DELIMITER //

CREATE PROCEDURE rapport_performance_quotidien(IN p_date DATE)
BEGIN
    -- Requêtes les plus lentes
    SELECT 'REQUETES LES PLUS LENTES' as section;
    SELECT 
        query_type,
        table_name,
        execution_time_ms,
        rows_examined,
        user_name,
        timestamp_execution
    FROM performance_monitoring
    WHERE DATE(timestamp_execution) = p_date
    ORDER BY execution_time_ms DESC
    LIMIT 10;
    
    -- Tables les plus sollicitées
    SELECT 'TABLES LES PLUS SOLLICITEES' as section;
    SELECT 
        table_name,
        COUNT(*) as nb_requetes,
        AVG(execution_time_ms) as temps_moyen,
        SUM(rows_examined) as total_lignes_examinees
    FROM performance_monitoring
    WHERE DATE(timestamp_execution) = p_date
      AND table_name IS NOT NULL
    GROUP BY table_name
    ORDER BY nb_requetes DESC
    LIMIT 10;
    
    -- Utilisateurs les plus actifs
    SELECT 'UTILISATEURS LES PLUS ACTIFS' as section;
    SELECT 
        user_name,
        COUNT(*) as nb_requetes,
        AVG(execution_time_ms) as temps_moyen
    FROM performance_monitoring
    WHERE DATE(timestamp_execution) = p_date
    GROUP BY user_name
    ORDER BY nb_requetes DESC
    LIMIT 10;
    
END //

DELIMITER ;

-- Test du système de monitoring
CALL log_performance('SELECT', 'SELECT * FROM commandes WHERE client_id = ?', 15, 100, 5, 'commandes');
CALL log_performance('UPDATE', 'UPDATE commandes SET statut = ? WHERE id = ?', 8, 1, 1, 'commandes');

-- Voir les statistiques
SELECT * FROM stats_performance WHERE date_exec = CURDATE();

-- Rapport quotidien
CALL rapport_performance_quotidien(CURDATE());
```
