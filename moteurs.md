# TP : Création de Tables avec Différents Moteurs (35 minutes)

#### 🎯 Objectifs
- Créer des tables avec différents moteurs (InnoDB, MyISAM, Memory)
- Tester les spécificités de chaque moteur
- Comparer les performances et fonctionnalités

---

#### 📋 **Partie A : Préparation de l'environnement** (5 min)

```sql
-- Créer une nouvelle base pour les tests
CREATE DATABASE test_moteurs;
USE test_moteurs;

-- Vérifier les moteurs disponibles
SHOW ENGINES;

-- Voir le moteur par défaut
SHOW VARIABLES LIKE 'default_storage_engine';
```

---

#### 📋 **Partie B : Table InnoDB (transactionnelle)** (10 min)

**Étape 1 : Créer une table InnoDB**
```sql
-- Table pour un système de commandes e-commerce
CREATE TABLE commandes_innodb (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    produit VARCHAR(100) NOT NULL,
    quantite INT NOT NULL CHECK (quantite > 0),
    prix_unitaire DECIMAL(8,2) NOT NULL CHECK (prix_unitaire > 0),
    date_commande TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    statut ENUM('EN_ATTENTE', 'CONFIRME', 'EXPEDIE', 'LIVRE') DEFAULT 'EN_ATTENTE',
    
    -- Index pour optimiser les recherches
    INDEX idx_client (client_id),
    INDEX idx_date (date_commande),
    INDEX idx_statut (statut)
) ENGINE=InnoDB;

-- Table des clients (pour les clés étrangères)
CREATE TABLE clients_innodb (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Ajouter la clé étrangère
ALTER TABLE commandes_innodb 
ADD FOREIGN KEY (client_id) REFERENCES clients_innodb(id);
```

**Étape 2 : Test des fonctionnalités InnoDB**
```sql
-- Insérer des clients
INSERT INTO clients_innodb (nom, email) VALUES
('Jean Dupont', 'jean.dupont@email.com'),
('Marie Martin', 'marie.martin@email.com'),
('Pierre Durand', 'pierre.durand@email.com');

-- Test de transaction avec ROLLBACK
START TRANSACTION;

INSERT INTO commandes_innodb (client_id, produit, quantite, prix_unitaire) VALUES
(1, 'Ordinateur portable', 1, 899.99),
(1, 'Souris sans fil', 1, 29.99),
(2, 'Clavier mécanique', 1, 129.99);

-- Vérifier les données temporaires
SELECT * FROM commandes_innodb;

-- Annuler la transaction
ROLLBACK;

-- Vérifier que les données ont été annulées
SELECT COUNT(*) as nb_commandes FROM commandes_innodb;
-- Résultat : 0

-- Transaction avec COMMIT
START TRANSACTION;

INSERT INTO commandes_innodb (client_id, produit, quantite, prix_unitaire) VALUES
(1, 'Ordinateur portable', 1, 899.99),
(2, 'Tablette', 1, 299.99);

COMMIT;

-- Vérifier les données persistantes
SELECT c.nom, cmd.produit, cmd.prix_unitaire 
FROM commandes_innodb cmd
JOIN clients_innodb c ON cmd.client_id = c.id;
```

**Étape 3 : Test des contraintes d'intégrité**
```sql
-- Test de clé étrangère (va échouer)
INSERT INTO commandes_innodb (client_id, produit, quantite, prix_unitaire) 
VALUES (999, 'Produit inexistant', 1, 10.00);

-- Test de contrainte CHECK (va échouer)
INSERT INTO commandes_innodb (client_id, produit, quantite, prix_unitaire) 
VALUES (1, 'Produit gratuit', 1, -5.00);
```

---

#### 📋 **Partie C : Table MyISAM (lecture intensive)** (10 min)

**Étape 1 : Créer une table MyISAM**
```sql
-- Table pour stocker des logs/statistiques
CREATE TABLE logs_myisam (
    id INT PRIMARY KEY AUTO_INCREMENT,
    date_log TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    niveau ENUM('INFO', 'backup_system', 'Sauvegarde quotidienne terminée avec succès', '127.0.0.1', 'mysqldump'),
('DEBUG', 'performance', 'Temps de réponse moyen: 0.25ms sur les 1000 dernières requêtes', '127.0.0.1', 'MySQL Monitor'),
('ERROR', 'web_server', 'Erreur 500 - Impossible de se connecter à la base de données', '192.168.1.200', 'Apache/2.4.41'),
('WARNING', 'security', 'Tentative de connexion depuis une IP suspecte bloquée', '203.0.113.1', 'Unknown'),
('INFO', 'application', 'Nouveau utilisateur enregistré avec succès', '192.168.1.150', 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36');

-- Vérifier l'insertion
SELECT COUNT(*) as total_logs FROM logs_myisam;
```

**Étape 3 : Test de la recherche FULLTEXT**
```sql
-- Recherche dans les messages contenant "connexion"
SELECT id, niveau, source, message, date_log
FROM logs_myisam
WHERE MATCH(message) AGAINST('connexion' IN NATURAL LANGUAGE MODE)
ORDER BY date_log DESC;

-- Recherche avec score de pertinence
SELECT id, niveau, source, message, 
       MATCH(message) AGAINST('base données' IN NATURAL LANGUAGE MODE) as score
FROM logs_myisam
WHERE MATCH(message) AGAINST('base données' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;

-- Recherche booléenne
SELECT id, niveau, source, message
FROM logs_myisam
WHERE MATCH(message, user_agent) AGAINST('+MySQL -Apache' IN BOOLEAN MODE);
```

**Étape 4 : Test des limitations MyISAM**
```sql
-- MyISAM ne supporte PAS les transactions
START TRANSACTION;

INSERT INTO logs_myisam (niveau, source, message) 
VALUES ('TEST', 'formation', 'Test transaction MyISAM');

-- Même avec ROLLBACK, l'insertion est persistante
ROLLBACK;

-- Vérifier que la ligne est bien présente
SELECT * FROM logs_myisam WHERE source = 'formation';

-- Nettoyer
DELETE FROM logs_myisam WHERE source = 'formation';
```

---

#### 📋 **Partie D : Table Memory (cache rapide)** (10 min)

**Étape 1 : Créer une table Memory**
```sql
-- Table pour gérer les sessions utilisateurs
CREATE TABLE sessions_memory (
    session_id VARCHAR(128) PRIMARY KEY,
    user_id INT NOT NULL,
    username VARCHAR(50),
    data TEXT,
    ip_address VARCHAR(45),
    derniere_activite TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Index pour recherches rapides
    INDEX idx_user (user_id),
    INDEX idx_username (username),
    INDEX idx_activite (derniere_activite)
) ENGINE=Memory;

-- Table de cache pour les données fréquemment consultées
CREATE TABLE cache_produits (
    id INT PRIMARY KEY,
    nom_produit VARCHAR(100),
    prix DECIMAL(8,2),
    stock INT,
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_nom (nom_produit)
) ENGINE=Memory;
```

**Étape 2 : Tests de performance**
```sql
-- Insérer des sessions de test
INSERT INTO sessions_memory (session_id, user_id, username, data, ip_address) VALUES
('sess_1a2b3c4d5e6f7g8h', 1, 'jean.dupont', '{"theme":"dark","lang":"fr"}', '192.168.1.100'),
('sess_9i8j7k6l5m4n3o2p', 2, 'marie.martin', '{"theme":"light","lang":"en"}', '192.168.1.101'),
('sess_q1w2e3r4t5y6u7i8', 3, 'pierre.durand', '{"theme":"auto","lang":"fr"}', '192.168.1.102'),
('sess_a9s8d7f6g5h4j3k2', 1, 'jean.dupont', '{"theme":"dark","lang":"es"}', '192.168.1.103');

-- Test de recherche rapide
SELECT session_id, username, data, ip_address
FROM sessions_memory
WHERE user_id = 1;

-- Mise à jour d'une session
UPDATE sessions_memory 
SET data = '{"theme":"light","lang":"fr","notifications":true}'
WHERE session_id = 'sess_1a2b3c4d5e6f7g8h';
```

**Étape 3 : Test du cache produits**
```sql
-- Simuler un cache de produits populaires
INSERT INTO cache_produits (id, nom_produit, prix, stock) VALUES
(1, 'iPhone 15 Pro', 1299.00, 50),
(2, 'Samsung Galaxy S24', 899.00, 75),
(3, 'MacBook Pro 14"', 2199.00, 25),
(4, 'Dell XPS 13', 1099.00, 40),
(5, 'iPad Air', 649.00, 60);

-- Simulation d'accès rapide au cache
SELECT nom_produit, prix, stock
FROM cache_produits
WHERE nom_produit LIKE '%Pro%'
ORDER BY prix DESC;

-- Test de performance avec BENCHMARK (optionnel)
SELECT BENCHMARK(100000, (SELECT COUNT(*) FROM cache_produits WHERE prix > 1000));
```

**Étape 4 : Test de la persistance**
```sql
-- Voir les données actuelles
SELECT COUNT(*) as sessions_actives FROM sessions_memory;
SELECT COUNT(*) as produits_cache FROM cache_produits;

-- ATTENTION : Simuler un redémarrage MySQL (uniquement si possible en formation)
-- Sur un système de test, vous pourriez faire :
-- sudo systemctl restart mysql

-- Après redémarrage, les tables Memory sont vides !
-- SELECT COUNT(*) FROM sessions_memory; -- Résultat : 0
-- SELECT COUNT(*) FROM cache_produits; -- Résultat : 0
```

**✅ Points à retenir :**
- **InnoDB** : ACID, clés étrangères, transactions
- **MyISAM** : Rapide en lecture, FULLTEXT, pas de transactions
- **Memory** : Ultra-rapide, volatile, limité par la RAM


## Validation TP - Moteurs
- [ ] Tables InnoDB avec clés étrangères et transactions testées
- [ ] Table MyISAM avec recherche FULLTEXT fonctionnelle
- [ ] Table Memory avec données volatiles comprise
- [ ] Spécificités de chaque moteur identifiées
