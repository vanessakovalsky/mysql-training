# TP 1 : SQL Procédural - Structures de Contrôle (45 minutes)

## 🎯 Objectifs
- Maîtriser les structures IF, CASE, WHILE, REPEAT
- Créer des procédures et fonctions utiles
- Comprendre la gestion d'erreurs


## 📋 **Partie A : Préparation de l'environnement** (10 min)

**Étape 1 : Créer la base de données de travail**
```sql
-- Se connecter en tant qu'utilisateur formation
mysql -u formation -p

-- Créer une nouvelle base pour les TP procéduraux
CREATE DATABASE formation_procedural;
USE formation_procedural;

-- Créer les tables de base
CREATE TABLE produits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    prix_ht DECIMAL(10,2) NOT NULL,
    stock INT DEFAULT 0,
    categorie ENUM('ELECTRONIQUE', 'VETEMENT', 'LIVRE', 'MAISON') NOT NULL,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE commandes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    numero_commande VARCHAR(50) UNIQUE,
    client_id INT NOT NULL,
    montant_ht DECIMAL(12,2) NOT NULL,
    statut ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE') DEFAULT 'NOUVELLE',
    date_commande TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE clients (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    type_client ENUM('PARTICULIER', 'PROFESSIONNEL') DEFAULT 'PARTICULIER',
    date_inscription TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

**Étape 2 : Insérer des données de test**
```sql
-- Insérer des produits
INSERT INTO produits (nom, prix_ht, stock, categorie) VALUES
('Smartphone Galaxy', 599.99, 25, 'ELECTRONIQUE'),
('T-shirt Coton Bio', 19.99, 100, 'VETEMENT'),
('Guide MySQL', 45.00, 50, 'LIVRE'),
('Lampe de Bureau LED', 89.99, 30, 'MAISON'),
('Ordinateur Portable', 899.99, 15, 'ELECTRONIQUE'),
('Jean Slim', 39.99, 75, 'VETEMENT');

-- Insérer des clients
INSERT INTO clients (nom, email, type_client) VALUES
('Jean Dupont', 'jean.dupont@email.com', 'PARTICULIER'),
('Marie Martin', 'marie.martin@email.com', 'PROFESSIONNEL'),
('Pierre Durand', 'pierre.durand@email.com', 'PARTICULIER'),
('Sophie Dubois', 'sophie.dubois@email.com', 'PROFESSIONNEL');

-- Insérer des commandes
INSERT INTO commandes (numero_commande, client_id, montant_ht, statut) VALUES
('CMD-20240301-001', 1, 619.98, 'LIVREE'),
('CMD-20240301-002', 2, 129.99, 'EXPEDIEE'),
('CMD-20240302-003', 3, 45.00, 'CONFIRMEE'),
('CMD-20240302-004', 4, 989.98, 'NOUVELLE');

-- Vérifier les données
SELECT 'Produits' as table_name, COUNT(*) as nb_lignes FROM produits
UNION ALL
SELECT 'Clients' as table_name, COUNT(*) as nb_lignes FROM clients
UNION ALL
SELECT 'Commandes' as table_name, COUNT(*) as nb_lignes FROM commandes;
```

## 📋 **Partie B : Fonctions avec structures conditionnelles** (15 min)

**Étape 1 : Fonction de calcul de remise avec IF**
```sql
DELIMITER //

CREATE FUNCTION calculer_remise_client(
    montant DECIMAL(10,2),
    type_client ENUM('PARTICULIER', 'PROFESSIONNEL')
)
RETURNS DECIMAL(5,2)
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE taux_remise DECIMAL(5,2);
    
    -- Remise selon le type de client et le montant
    IF type_client = 'PROFESSIONNEL' THEN
        IF montant >= 1000 THEN
            SET taux_remise = 0.20;  -- 20% pour pro > 1000€
        ELSEIF montant >= 500 THEN
            SET taux_remise = 0.15;  -- 15% pour pro > 500€
        ELSE
            SET taux_remise = 0.10;  -- 10% pour pro
        END IF;
    ELSE -- PARTICULIER
        IF montant >= 1000 THEN
            SET taux_remise = 0.15;  -- 15% pour particulier > 1000€
        ELSEIF montant >= 200 THEN
            SET taux_remise = 0.05;  -- 5% pour particulier > 200€
        ELSE
            SET taux_remise = 0.00;  -- Pas de remise
        END IF;
    END IF;
    
    RETURN taux_remise;
END //

DELIMITER ;

-- Test de la fonction
SELECT 
    c.nom,
    c.type_client,
    cmd.montant_ht,
    calculer_remise_client(cmd.montant_ht, c.type_client) * 100 as remise_pct,
    ROUND(cmd.montant_ht * calculer_remise_client(cmd.montant_ht, c.type_client), 2) as remise_euros
FROM commandes cmd
JOIN clients c ON cmd.client_id = c.id;
```

**Étape 2 : Fonction avec CASE pour conversion de statut**
```sql
DELIMITER //

CREATE FUNCTION statut_lisible(statut_code ENUM('NOUVELLE', 'CONFIRMEE', 'EXPEDIEE', 'LIVREE', 'ANNULEE'))
RETURNS VARCHAR(30)
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE statut_fr VARCHAR(30);
    
    SET statut_fr = CASE statut_code
        WHEN 'NOUVELLE' THEN '📝 Nouvelle commande'
        WHEN 'CONFIRMEE' THEN '✅ Confirmée'
        WHEN 'EXPEDIEE' THEN '🚚 Expédiée'
        WHEN 'LIVREE' THEN '📦 Livrée'
        WHEN 'ANNULEE' THEN '❌ Annulée'
        ELSE '❓ Statut inconnu'
    END;
    
    RETURN statut_fr;
END //

DELIMITER ;

-- Test de la fonction
SELECT 
    numero_commande,
    statut,
    statut_lisible(statut) as statut_français,
    montant_ht
FROM commandes
ORDER BY date_commande;
```

**✅ Point de contrôle :** Les fonctions retournent les bonnes valeurs selon les conditions.


## 📋 **Partie C : Procédures avec boucles** (20 min)

**Étape 1 : Procédure avec boucle WHILE pour générer des données**
```sql
DELIMITER //

CREATE PROCEDURE generer_commandes_test(
    IN nb_commandes INT,
    OUT commandes_creees INT,
    OUT message VARCHAR(255)
)
BEGIN
    DECLARE compteur INT DEFAULT 1;
    DECLARE client_aleatoire INT;
    DECLARE montant_aleatoire DECIMAL(10,2);
    DECLARE numero_cmd VARCHAR(50);
    
    SET commandes_creees = 0;
    
    -- Vérification des paramètres
    IF nb_commandes <= 0 OR nb_commandes > 100 THEN
        SET message = 'ERREUR: Le nombre de commandes doit être entre 1 et 100';
        SET commandes_creees = 0;
    ELSE
        -- Boucle de création
        WHILE compteur <= nb_commandes DO
            -- Sélectionner un client aléatoire
            SELECT id INTO client_aleatoire 
            FROM clients 
            ORDER BY RAND() 
            LIMIT 1;
            
            -- Générer un montant aléatoire entre 20 et 500
            SET montant_aleatoire = ROUND(20 + RAND() * 480, 2);
            
            -- Générer le numéro de commande
            SET numero_cmd = CONCAT('TEST-', DATE_FORMAT(NOW(), '%Y%m%d'), '-', LPAD(compteur, 3, '0'));
            
            -- Insérer la commande
            INSERT INTO commandes (numero_commande, client_id, montant_ht, statut)
            VALUES (numero_cmd, client_aleatoire, montant_aleatoire, 'NOUVELLE');
            
            SET compteur = compteur + 1;
            SET commandes_creees = commandes_creees + 1;
        END WHILE;
        
        SET message = CONCAT('SUCCESS: ', commandes_creees, ' commandes créées avec succès');
    END IF;
    
END //

DELIMITER ;

-- Test de la procédure
CALL generer_commandes_test(5, @nb_creees, @msg);
SELECT @nb_creees as commandes_créées, @msg as message;

-- Vérifier les nouvelles commandes
SELECT numero_commande, client_id, montant_ht, statut 
FROM commandes 
WHERE numero_commande LIKE 'TEST-%'
ORDER BY date_commande DESC;
```

**Étape 2 : Procédure avec REPEAT pour calcul itératif**
```sql
DELIMITER //

CREATE PROCEDURE calculer_fibonacci(
    IN nombre_termes INT,
    OUT sequence_fibonacci TEXT
)
BEGIN
    DECLARE compteur INT DEFAULT 1;
    DECLARE terme_actuel BIGINT DEFAULT 1;
    DECLARE terme_precedent BIGINT DEFAULT 0;
    DECLARE terme_suivant BIGINT;
    DECLARE sequence_temp TEXT DEFAULT '';
    
    -- Vérification du paramètre
    IF nombre_termes <= 0 THEN
        SET sequence_fibonacci = 'ERREUR: Le nombre de termes doit être positif';
    ELSE
        -- Cas particuliers
        IF nombre_termes = 1 THEN
            SET sequence_fibonacci = '0';
        ELSEIF nombre_termes = 2 THEN
            SET sequence_fibonacci = '0, 1';
        ELSE
            SET sequence_temp = '0, 1';
            SET compteur = 3;  -- On commence au 3ème terme
            
            REPEAT
                SET terme_suivant = terme_precedent + terme_actuel;
                SET sequence_temp = CONCAT(sequence_temp, ', ', terme_suivant);
                
                -- Préparer pour le terme suivant
                SET terme_precedent = terme_actuel;
                SET terme_actuel = terme_suivant;
                SET compteur = compteur + 1;
                
            UNTIL compteur > nombre_termes END REPEAT;
            
            SET sequence_fibonacci = sequence_temp;
        END IF;
    END IF;
    
END //

DELIMITER ;

-- Test de la procédure
CALL calculer_fibonacci(10, @fib_sequence);
SELECT @fib_sequence as 'Séquence de Fibonacci (10 termes)';

CALL calculer_fibonacci(15, @fib_sequence);
SELECT @fib_sequence as 'Séquence de Fibonacci (15 termes)';
```

## **Points de Contrôle**

- [ ] Fonction avec IF/ELSEIF/ELSE fonctionnelle
- [ ] Fonction avec CASE fonctionnelle  
- [ ] Procédure avec boucle WHILE fonctionnelle
- [ ] Procédure avec boucle REPEAT fonctionnelle
- [ ] Gestion des paramètres IN/OUT/INOUT maîtrisée
