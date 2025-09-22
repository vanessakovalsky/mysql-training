### Défi 1 : Système de Cache Intelligent avec Triggers

#### 🎯 Objectif
Créer un système de cache automatique qui se met à jour via des triggers.

```sql
-- Table de cache pour les statistiques client
CREATE TABLE cache_stats_clients (
    client_id INT PRIMARY KEY,
    nb_commandes INT DEFAULT 0,
    montant_total DECIMAL(12,2) DEFAULT 0,
    montant_moyen DECIMAL(10,2) DEFAULT 0,
    derniere_commande DATE,
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (client_id) REFERENCES clients(id)
) ENGINE=InnoDB;

-- Procédure pour recalculer le cache d'un client
DELIMITER //

CREATE PROCEDURE recalculer_cache_client(IN p_client_id INT)
BEGIN
    DECLARE v_nb_commandes INT DEFAULT 0;
    DECLARE v_montant_total DECIMAL(12,2) DEFAULT 0;
    DECLARE v_montant_moyen DECIMAL(10,2) DEFAULT 0;
    DECLARE v_derniere_commande DATE;
    
    -- Calculer les statistiques
    SELECT 
        COUNT(*),
        COALESCE(SUM(montant_ht), 0),
        COALESCE(AVG(montant_ht), 0),
        MAX(DATE(date_commande))
    INTO v_nb_commandes, v_montant_total, v_montant_moyen, v_derniere_commande
    FROM commandes 
    WHERE client_id = p_client_id AND statut != 'ANNULEE';
    
    -- Mettre à jour ou insérer dans le cache
    INSERT INTO cache_stats_clients 
    (client_id, nb_commandes, montant_total, montant_moyen, derniere_commande)
    VALUES (p_client_id, v_nb_commandes, v_montant_total, v_montant_moyen, v_derniere_commande)
    ON DUPLICATE KEY UPDATE
        nb_commandes = VALUES(nb_commandes),
        montant_total = VALUES(montant_total),
        montant_moyen = VALUES(montant_moyen),
        derniere_commande = VALUES(derniere_commande);
END //

DELIMITER ;

-- Trigger pour mettre à jour le cache automatiquement
DELIMITER //

CREATE TRIGGER maj_cache_stats_client
    AFTER INSERT ON commandes
    FOR EACH ROW
BEGIN
    CALL recalculer_cache_client(NEW.client_id);
END //

CREATE TRIGGER maj_cache_stats_client_update
    AFTER UPDATE ON commandes
    FOR EACH ROW
BEGIN
    -- Si le client a changé, recalculer pour les deux
    IF OLD.client_id != NEW.client_id THEN
        CALL recalculer_cache_client(OLD.client_id);
        CALL recalculer_cache_client(NEW.client_id);
    ELSE
        CALL recalculer_cache_client(NEW.client_id);
    END IF;
END //

CREATE TRIGGER maj_cache_stats_client_delete
    AFTER DELETE ON commandes
    FOR EACH ROW
BEGIN
    CALL recalculer_cache_client(OLD.client_id);
END //

DELIMITER ;

-- Test du système de cache
INSERT INTO commandes (client_id, montant_ht, numero_commande)
VALUES (1, 250.00, 'CACHE-TEST-001');

-- Vérifier que le cache s'est mis à jour
SELECT 
    c.nom,
    cs.nb_commandes,
    cs.montant_total,
    cs.montant_moyen,
    cs.derniere_commande
FROM clients c
JOIN cache_stats_clients cs ON c.id = cs.client_id
WHERE c.id = 1;
```
