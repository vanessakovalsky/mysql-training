### Défi 2 : Système de Workflow avec États

#### 🎯 Objectif
Créer un système de workflow pour les commandes avec validation des transitions d'états.

```sql
-- Table des transitions d'états autorisées
CREATE TABLE workflow_transitions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    etat_source ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE'),
    etat_destination ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE'),
    condition_requise VARCHAR(255),
    action_auto VARCHAR(255),
    UNIQUE KEY unique_transition (etat_source, etat_destination)
) ENGINE=InnoDB;

-- Insérer les transitions autorisées
INSERT INTO workflow_transitions (etat_source, etat_destination, condition_requise, action_auto) VALUES
('NOUVELLE', 'CONFIRMEE', 'montant > 0', 'generer_numero_suivi'),
('NOUVELLE', 'ANNULEE', NULL, 'liberer_stock'),
('CONFIRMEE', 'EXPEDIEE', 'stock_disponible', 'reserver_stock'),
('CONFIRMEE', 'ANNULEE', NULL, 'liberer_stock'),
('EXPEDIEE', 'LIVREE', NULL, 'confirmer_livraison'),
('EXPEDIEE', 'ANNULEE', 'delai_expedition_depasse', 'rembourser_client');

-- Table d'historique des changements d'état
CREATE TABLE historique_workflow (
    id INT PRIMARY KEY AUTO_INCREMENT,
    commande_id INT NOT NULL,
    etat_precedent ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE'),
    nouvel_etat ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE'),
    utilisateur VARCHAR(100),
    commentaire TEXT,
    timestamp_changement TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (commande_id) REFERENCES commandes(id)
) ENGINE=InnoDB;

-- Procédure pour changer l'état d'une commande
DELIMITER //

CREATE PROCEDURE changer_etat_commande(
    IN p_commande_id INT,
    IN p_nouvel_etat ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE'),
    IN p_commentaire TEXT,
    OUT p_resultat VARCHAR(255)
)
BEGIN
    DECLARE v_etat_actuel ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE');
    DECLARE v_transition_autorisee INT DEFAULT 0;
    DECLARE v_montant DECIMAL(12,2);
    DECLARE v_condition VARCHAR(255);
    DECLARE v_action VARCHAR(255);
    
    -- Handler pour les erreurs
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_resultat = 'ERREUR: Impossible de changer l\'état de la commande';
    END;
    
    START TRANSACTION;
    
    -- Récupérer l'état actuel de la commande
    SELECT statut, montant_ht 
    INTO v_etat_actuel, v_montant
    FROM commandes 
    WHERE id = p_commande_id;
    
    IF v_etat_actuel IS NULL THEN
        SET p_resultat = 'ERREUR: Commande non trouvée';
        ROLLBACK;
    ELSEIF v_etat_actuel = p_nouvel_etat THEN
        SET p_resultat = 'INFO: La commande est déjà dans cet état';
        ROLLBACK;
    ELSE
        -- Vérifier si la transition est autorisée
        SELECT COUNT(*), condition_requise, action_auto
        INTO v_transition_autorisee, v_condition, v_action
        FROM workflow_transitions 
        WHERE etat_source = v_etat_actuel AND etat_destination = p_nouvel_etat;
        
        IF v_transition_autorisee = 0 THEN
            SET p_resultat = CONCAT('ERREUR: Transition non autorisée de ', v_etat_actuel, ' vers ', p_nouvel_etat);
            ROLLBACK;
        ELSE
            -- Vérifier les conditions (simplifiée pour l'exemple)
            IF v_condition IS NOT NULL AND v_condition = 'montant > 0' AND v_montant <= 0 THEN
                SET p_resultat = 'ERREUR: Condition non remplie - montant doit être positif';
                ROLLBACK;
            ELSE
                -- Effectuer le changement d'état
                UPDATE commandes SET statut = p_nouvel_etat WHERE id = p_commande_id;
                
                -- Enregistrer dans l'historique
                INSERT INTO historique_workflow 
                (commande_id, etat_precedent, nouvel_etat, utilisateur, commentaire)
                VALUES (p_commande_id, v_etat_actuel, p_nouvel_etat, USER(), p_commentaire);
                
                -- Exécuter l'action automatique (simulation)
                IF v_action IS NOT NULL THEN
                    INSERT INTO audit_operations 
                    (table_name, operation_type, record_id, new_values, user_name)
                    VALUES ('workflow', 'ACTION', p_commande_id, 
                           JSON_OBJECT('action', v_action, 'etat', p_nouvel_etat), USER());
                END IF;
                
                COMMIT;
                SET p_resultat = CONCAT('SUCCESS: État changé de ', v_etat_actuel, ' vers ', p_nouvel_etat);
            END IF;
        END IF;
    END IF;
    
END //

DELIMITER ;

-- Test du workflow
CALL changer_etat_commande(1, 'CONFIRMEE', 'Commande validée par l\'équipe', @resultat);
SELECT @resultat as résultat;

-- Voir l'historique
SELECT 
    h.commande_id,
    c.numero_commande,
    h.etat_precedent,
    h.nouvel_etat,
    h.commentaire,
    h.timestamp_changement
FROM historique_workflow h
JOIN commandes c ON h.commande_id = c.id
ORDER BY h.timestamp_changement DESC;
```
