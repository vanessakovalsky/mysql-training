# TP  Curseurs et Déclencheurs (40 minutes)

## 🎯 Objectifs
- Utiliser les curseurs pour parcourir des résultats
- Créer des triggers d'audit et de validation
- Comprendre l'ordre d'exécution des triggers

## 📋 **Partie A : Curseurs avancés** (20 min)

**Étape 1 : Créer des tables pour l'audit**
```sql
-- Table pour stocker les statistiques de ventes
CREATE TABLE stats_ventes_mensuelles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    annee INT NOT NULL,
    mois INT NOT NULL,
    nb_commandes INT DEFAULT 0,
    montant_total DECIMAL(15,2) DEFAULT 0,
    montant_moyen DECIMAL(10,2) DEFAULT 0,
    date_calcul TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY unique_mois (annee, mois)
) ENGINE=InnoDB;

-- Table d'audit détaillée
CREATE TABLE audit_operations (
    id INT PRIMARY KEY AUTO_INCREMENT,
    table_name VARCHAR(50),
    operation_type ENUM('INSERT', 'UPDATE', 'DELETE'),
    record_id INT,
    old_values JSON,
    new_values JSON,
    user_name VARCHAR(100),
    timestamp_operation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

**Étape 2 : Procédure avec curseur pour calculer les statistiques**
```sql
DELIMITER //

CREATE PROCEDURE calculer_stats_mensuelles(
    IN annee_cible INT,
    OUT mois_traites INT,
    OUT message_resultat VARCHAR(500)
)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE mois_courant INT;
    DECLARE nb_commandes_mois INT;
    DECLARE montant_total_mois DECIMAL(15,2);
    DECLARE montant_moyen_mois DECIMAL(10,2);
    DECLARE message_temp VARCHAR(255);
    
    -- Curseur pour parcourir les mois avec des commandes
    DECLARE curseur_mois CURSOR FOR
        SELECT 
            MONTH(date_commande) as mois,
            COUNT(*) as nb_commandes,
            SUM(montant_ht) as montant_total,
            AVG(montant_ht) as montant_moyen
        FROM commandes
        WHERE YEAR(date_commande) = annee_cible
          AND statut != 'ANNULEE'
        GROUP BY MONTH(date_commande)
        ORDER BY MONTH(date_commande);
    
    -- Handler pour la fin du curseur
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    SET mois_traites = 0;
    SET message_resultat = '';
    
    -- Ouvrir le curseur
    OPEN curseur_mois;
    
    -- Parcourir les résultats
    lecture_mois: LOOP
        FETCH curseur_mois INTO mois_courant, nb_commandes_mois, montant_total_mois, montant_moyen_mois;
        
        IF done THEN
            LEAVE lecture_mois;
        END IF;
        
        -- Insérer ou mettre à jour les statistiques
        INSERT INTO stats_ventes_mensuelles 
        (annee, mois, nb_commandes, montant_total, montant_moyen)
        VALUES (annee_cible, mois_courant, nb_commandes_mois, montant_total_mois, montant_moyen_mois)
        ON DUPLICATE KEY UPDATE
            nb_commandes = VALUES(nb_commandes),
            montant_total = VALUES(montant_total),
            montant_moyen = VALUES(montant_moyen),
            date_calcul = NOW();
        
        -- Construire le message de résultat
        SET message_temp = CONCAT(
            'Mois ', mois_courant, ': ', nb_commandes_mois, ' commandes, ',
            FORMAT(montant_total_mois, 2), '€ total; '
        );
        SET message_resultat = CONCAT(message_resultat, message_temp);
        
        SET mois_traites = mois_traites + 1;
        
    END LOOP;
    
    -- Fermer le curseur
    CLOSE curseur_mois;
    
    IF mois_traites = 0 THEN
        SET message_resultat = CONCAT('Aucune commande trouvée pour l\'année ', annee_cible);
    ELSE
        SET message_resultat = CONCAT(
            'Stats calculées pour ', mois_traites, ' mois de ', annee_cible, ': ',
            message_resultat
        );
    END IF;
    
END //

DELIMITER ;

-- Test de la procédure
CALL calculer_stats_mensuelles(2024, @nb_mois, @message);
SELECT @nb_mois as mois_traités, @message as résultat;

-- Vérifier les statistiques générées
SELECT 
    annee,
    mois,
    nb_commandes,
    FORMAT(montant_total, 2) as montant_total,
    FORMAT(montant_moyen, 2) as panier_moyen
FROM stats_ventes_mensuelles
ORDER BY annee, mois;
```

## 📋 **Partie B : Déclencheurs (Triggers)** (20 min)

**Étape 1 : Trigger d'audit complet**
```sql
-- Trigger AFTER INSERT sur commandes
DELIMITER //

CREATE TRIGGER audit_commande_insert
    AFTER INSERT ON commandes
    FOR EACH ROW
BEGIN
    INSERT INTO audit_operations (
        table_name, 
        operation_type, 
        record_id, 
        new_values, 
        user_name
    )
    VALUES (
        'commandes',
        'INSERT',
        NEW.id,
        JSON_OBJECT(
            'numero_commande', NEW.numero_commande,
            'client_id', NEW.client_id,
            'montant_ht', NEW.montant_ht,
            'statut', NEW.statut,
            'date_commande', NEW.date_commande
        ),
        USER()
    );
END //

DELIMITER ;

-- Trigger AFTER UPDATE sur commandes
DELIMITER //

CREATE TRIGGER audit_commande_update
    AFTER UPDATE ON commandes
    FOR EACH ROW
BEGIN
    INSERT INTO audit_operations (
        table_name, 
        operation_type, 
        record_id, 
        old_values,
        new_values, 
        user_name
    )
    VALUES (
        'commandes',
        'UPDATE',
        NEW.id,
        JSON_OBJECT(
            'numero_commande', OLD.numero_commande,
            'client_id', OLD.client_id,
            'montant_ht', OLD.montant_ht,
            'statut', OLD.statut
        ),
        JSON_OBJECT(
            'numero_commande', NEW.numero_commande,
            'client_id', NEW.client_id,
            'montant_ht', NEW.montant_ht,
            'statut', NEW.statut
        ),
        USER()
    );
END //

DELIMITER ;
```

**Étape 2 : Trigger de validation BEFORE**
```sql
-- Trigger de validation avant insertion
DELIMITER //

CREATE TRIGGER valider_commande_before_insert
    BEFORE INSERT ON commandes
    FOR EACH ROW
BEGIN
    -- Validation du montant
    IF NEW.montant_ht <= 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Le montant de la commande doit être positif';
    END IF;
    
    -- Validation de l'existence du client
    IF NOT EXISTS (SELECT 1 FROM clients WHERE id = NEW.client_id) THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Le client spécifié n\'existe pas';
    END IF;
    
    -- Auto-génération du numéro de commande si vide
    IF NEW.numero_commande IS NULL OR NEW.numero_commande = '' THEN
        SET NEW.numero_commande = CONCAT(
            'AUTO-', 
            DATE_FORMAT(NOW(), '%Y%m%d'), 
            '-',
            LPAD(LAST_INSERT_ID() + 1, 4, '0')
        );
    END IF;
    
    -- Validation du format du numéro de commande
    IF NEW.numero_commande NOT REGEXP '^[A-Z0-9-]+ THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Le numéro de commande ne doit contenir que des lettres, chiffres et tirets';
    END IF;
    
END //

DELIMITER ;

-- Trigger de mise à jour des stocks (simulation)
DELIMITER //

CREATE TRIGGER maj_stock_commande
    AFTER UPDATE ON commandes
    FOR EACH ROW
BEGIN
    -- Simulation : si le statut passe à CONFIRMEE, on pourrait mettre à jour les stocks
    IF OLD.statut != 'CONFIRMEE' AND NEW.statut = 'CONFIRMEE' THEN
        -- Dans un vrai système, on aurait une table lignes_commande
        -- et on mettrait à jour le stock de chaque produit
        INSERT INTO audit_operations (
            table_name, 
            operation_type, 
            record_id, 
            new_values, 
            user_name
        )
        VALUES (
            'stocks',
            'UPDATE',
            NEW.id,
            JSON_OBJECT(
                'action', 'RESERVATION_STOCK',
                'commande_id', NEW.id,
                'montant', NEW.montant_ht
            ),
            USER()
        );
    END IF;
END //

DELIMITER ;
```

**Étape 3 : Test des triggers**
```sql
-- Test 1 : Insertion normale (doit créer un audit)
INSERT INTO commandes (client_id, montant_ht, statut)
VALUES (1, 156.75, 'NOUVELLE');

-- Test 2 : Insertion avec erreur (montant négatif)
-- Cette commande doit échouer
INSERT INTO commandes (client_id, montant_ht, statut)
VALUES (1, -50.00, 'NOUVELLE');

-- Test 3 : Insertion avec client inexistant
-- Cette commande doit échouer
INSERT INTO commandes (client_id, montant_ht, statut)
VALUES (999, 100.00, 'NOUVELLE');

-- Test 4 : Mise à jour du statut (doit créer un audit)
UPDATE commandes 
SET statut = 'CONFIRMEE' 
WHERE numero_commande LIKE 'AUTO-%' 
LIMIT 1;

-- Vérifier les audits créés
SELECT 
    table_name,
    operation_type,
    record_id,
    new_values,
    user_name,
    timestamp_operation
FROM audit_operations
ORDER BY timestamp_operation DESC
LIMIT 5;
```

## Checklist de Validation Finale


**Curseurs :**
- [ ] Déclaration et ouverture de curseur réussies
- [ ] Parcours complet avec LOOP/FETCH
- [ ] Handler NOT FOUND implémenté
- [ ] Fermeture de curseur effectuée
- [ ] Traitement des données ligne par ligne fonctionnel

**Triggers :**
- [ ] Trigger BEFORE INSERT avec validation
- [ ] Trigger AFTER UPDATE pour audit
- [ ] Utilisation de OLD/NEW appropriée
- [ ] SIGNAL pour erreurs personnalisées
- [ ] JSON utilisé pour stockage d'audit

