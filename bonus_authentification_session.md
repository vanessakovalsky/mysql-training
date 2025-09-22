### Défi 3 : Système d'Authentification et Sessions Avancé

#### 🎯 Objectif
Créer un système complet de gestion des sessions utilisateurs avec sécurité renforcée.

```sql
-- Table des sessions utilisateurs
CREATE TABLE user_sessions (
    session_id VARCHAR(128) PRIMARY KEY,
    user_id INT NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    login_method ENUM('PASSWORD', 'TOKEN', 'SSO') DEFAULT 'PASSWORD',
    INDEX idx_user_id (user_id),
    INDEX idx_expires (expires_at),
    INDEX idx_active (is_active)
) ENGINE=InnoDB;

-- Table des tentatives de connexion
CREATE TABLE login_attempts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100),
    ip_address VARCHAR(45),
    success BOOLEAN DEFAULT FALSE,
    failure_reason VARCHAR(255),
    attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_ip_time (ip_address, attempted_at),
    INDEX idx_username_time (username, attempted_at)
) ENGINE=InnoDB;

-- Procédure de création de session
DELIMITER //

CREATE PROCEDURE creer_session_utilisateur(
    IN p_user_id INT,
    IN p_ip_address VARCHAR(45),
    IN p_user_agent TEXT,
    IN p_duree_heures INT,
    OUT p_session_id VARCHAR(128),
    OUT p_message VARCHAR(255)
)
BEGIN
    DECLARE v_user_exists INT DEFAULT 0;
    DECLARE v_max_sessions INT DEFAULT 5;
    DECLARE v_sessions_actives INT DEFAULT 0;
    
    -- Vérifier l'existence de l'utilisateur
    SELECT COUNT(*) INTO v_user_exists FROM clients WHERE id = p_user_id;
    
    IF v_user_exists = 0 THEN
        SET p_session_id = NULL;
        SET p_message = 'ERREUR: Utilisateur non trouvé';
    ELSE
        -- Compter les sessions actives pour cet utilisateur
        SELECT COUNT(*) INTO v_sessions_actives 
        FROM user_sessions 
        WHERE user_id = p_user_id AND is_active = TRUE AND expires_at > NOW();
        
        IF v_sessions_actives >= v_max_sessions THEN
            -- Supprimer la plus ancienne session
            DELETE FROM user_sessions 
            WHERE user_id = p_user_id 
            ORDER BY last_activity ASC 
            LIMIT 1;
        END IF;
        
        -- Générer un ID de session unique
        SET p_session_id = CONCAT(
            HEX(RANDOM_BYTES(16)),
            UNIX_TIMESTAMP(),
            HEX(RANDOM_BYTES(8))
        );
        
        -- Créer la nouvelle session
        INSERT INTO user_sessions 
        (session_id, user_id, ip_address, user_agent, expires_at)
        VALUES 
        (p_session_id, p_user_id, p_ip_address, p_user_agent, 
         DATE_ADD(NOW(), INTERVAL p_duree_heures HOUR));
        
        SET p_message = 'SUCCESS: Session créée avec succès';
    END IF;
    
END //

DELIMITER ;

-- Procédure de validation de session
DELIMITER //

CREATE PROCEDURE valider_session(
    IN p_session_id VARCHAR(128),
    IN p_ip_address VARCHAR(45),
    OUT p_user_id INT,
    OUT p_valide BOOLEAN,
    OUT p_message VARCHAR(255)
)
BEGIN
    DECLARE v_user_id INT;
    DECLARE v_expires_at TIMESTAMP;
    DECLARE v_ip_stored VARCHAR(45);
    DECLARE v_is_active BOOLEAN;
    
    SET p_valide = FALSE;
    SET p_user_id = NULL;
    
    -- Récupérer les informations de session
    SELECT user_id, expires_at, ip_address, is_active
    INTO v_user_id, v_expires_at, v_ip_stored, v_is_active
    FROM user_sessions 
    WHERE session_id = p_session_id;
    
    IF v_user_id IS NULL THEN
        SET p_message = 'ERREUR: Session non trouvée';
    ELSEIF NOT v_is_active THEN
        SET p_message = 'ERREUR: Session désactivée';
    ELSEIF v_expires_at < NOW() THEN
        -- Marquer la session comme expirée
        UPDATE user_sessions SET is_active = FALSE WHERE session_id = p_session_id;
        SET p_message = 'ERREUR: Session expirée';
    ELSEIF v_ip_stored != p_ip_address THEN
        -- Sécurité : vérification IP (optionnel selon la politique)
        SET p_message = 'ATTENTION: Changement d\'adresse IP détecté';
        -- On peut choisir de valider quand même ou non selon la politique
        SET p_valide = TRUE;
        SET p_user_id = v_user_id;
        
        -- Mettre à jour la dernière activité
        UPDATE user_sessions SET last_activity = NOW() WHERE session_id = p_session_id;
    ELSE
        SET p_valide = TRUE;
        SET p_user_id = v_user_id;
        SET p_message = 'SUCCESS: Session valide';
        
        -- Mettre à jour la dernière activité
        UPDATE user_sessions SET last_activity = NOW() WHERE session_id = p_session_id;
    END IF;
    
END //

DELIMITER ;

-- Procédure de nettoyage des sessions expirées
DELIMITER //

CREATE PROCEDURE nettoyer_sessions_expirees()
BEGIN
    DECLARE v_sessions_supprimees INT;
    
    -- Supprimer les sessions expirées
    DELETE FROM user_sessions 
    WHERE expires_at < NOW() OR (last_activity < DATE_SUB(NOW(), INTERVAL 7 DAY));
    
    SET v_sessions_supprimees = ROW_COUNT();
    
    -- Enregistrer l'action de nettoyage
    INSERT INTO audit_operations 
    (table_name, operation_type, record_id, new_values, user_name)
    VALUES ('user_sessions', 'CLEANUP', 0, 
           JSON_OBJECT('sessions_supprimees', v_sessions_supprimees), 'SYSTEM');
    
    SELECT CONCAT(v_sessions_supprimees, ' sessions expirées supprimées') as resultat;
    
END //

DELIMITER ;

-- Event automatique pour le nettoyage (si les events sont activés)
-- CREATE EVENT IF NOT EXISTS cleanup_sessions
-- ON SCHEDULE EVERY 1 HOUR
-- DO CALL nettoyer_sessions_expirees();

-- Tests du système de sessions
CALL creer_session_utilisateur(1, '192.168.1.100', 'Mozilla/5.0...', 24, @session_id, @message);
SELECT @session_id as session_créée, @message as message;

CALL valider_session(@session_id, '192.168.1.100', @user_id, @valide, @message);
SELECT @user_id as user_id, @valide as session_valide, @message as message;
```
