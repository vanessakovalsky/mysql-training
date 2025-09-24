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
    niveau ENUM('INFO', 'WARNING', 'ERROR', 'DEBUG') DEFAULT 'INFO',
    source VARCHAR(50) NOT NULL,
    message TEXT NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    session_id VARCHAR(64),
    
    -- Index pour recherches rapides
    INDEX idx_date (date_log),
    INDEX idx_niveau (niveau),
    INDEX idx_source (source),
    INDEX idx_session (session_id),
    
    -- Index FULLTEXT pour recherche textuelle (spécificité MyISAM)
    FULLTEXT KEY ft_message (message),
    FULLTEXT KEY ft_user_agent (user_agent),
    FULLTEXT KEY ft_message_agent (message, user_agent)
    
) ENGINE=MyISAM COMMENT='Table de logs avec recherche full-text';

-- Vérifier la structure
SHOW CREATE TABLE logs_myisam\G;

```

**Étape 2 : Insérer des données de test dans MyISAM**

```sql
-- Insérer des logs de test variés
INSERT INTO logs_myisam (niveau, source, message, ip_address, user_agent, session_id) VALUES
('INFO', 'web_server', 'Connexion utilisateur réussie pour user123', '192.168.1.100', 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36', 'sess_abc123'),
('WARNING', 'database', 'Requête lente détectée: SELECT * FROM large_table WHERE col LIKE "%pattern%"', '127.0.0.1', 'MySQL Workbench 8.0', 'sess_db001'),
('ERROR', 'auth_system', 'Tentative de connexion échouée - mot de passe incorrect pour admin', '192.168.1.50', 'curl/7.68.0', 'sess_hack01'),
('INFO', 'backup_system', 'Sauvegarde quotidienne terminée avec succès - 2.3GB sauvegardés', '127.0.0.1', 'mysqldump/8.0.28', 'sess_backup'),
('DEBUG', 'performance', 'Temps de réponse moyen: 0.25ms sur les 1000 dernières requêtes', '127.0.0.1', 'MySQL Monitor v2.1', 'sess_perf01'),
('ERROR', 'web_server', 'Erreur 500 - Impossible de se connecter à la base de données MySQL', '192.168.1.200', 'Apache/2.4.41 (Ubuntu) PHP/7.4.3', 'sess_web500'),
('WARNING', 'security', 'Tentative de connexion depuis une IP suspecte bloquée automatiquement', '203.0.113.1', 'Unknown/Suspicious', 'sess_sec001'),
('INFO', 'application', 'Nouveau utilisateur enregistré avec succès - ID: 1547', '192.168.1.150', 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36', 'sess_reg123'),
('ERROR', 'payment', 'Échec du paiement - carte expirée pour transaction T789456', '192.168.1.175', 'Mobile App v3.2.1', 'sess_pay789'),
('INFO', 'cache', 'Cache vidé avec succès - 1247 entrées supprimées', '127.0.0.1', 'Redis Cache Manager', 'sess_cache1'),
('WARNING', 'disk_space', 'Espace disque faible sur /var/log - 85% utilisé', '127.0.0.1', 'System Monitor', 'sess_disk01'),
('DEBUG', 'api', 'Appel API réussi vers service externe - latence 142ms', '192.168.1.180', 'Internal API Client v1.0', 'sess_api001');

-- Vérifier les données insérées
SELECT COUNT(*) as total_logs FROM logs_myisam;

-- Voir quelques exemples
SELECT niveau, source, LEFT(message, 60) as message_extrait, ip_address
FROM logs_myisam 
ORDER BY date_log DESC 
LIMIT 5;
```

**Étape 3 : Test de la recherche FULLTEXT**
```sql
-- Test 1: Recherche dans les messages contenant "connexion"
SELECT 
    niveau,
    source,
    message,
    ip_address,
    date_log
FROM logs_myisam
WHERE MATCH(message) AGAINST('connexion' IN NATURAL LANGUAGE MODE)
ORDER BY date_log DESC;

-- Test 2: Recherche avec score de pertinence
SELECT 
    niveau,
    source,
    LEFT(message, 80) as message,
    MATCH(message) AGAINST('base données MySQL' IN NATURAL LANGUAGE MODE) as score
FROM logs_myisam
WHERE MATCH(message) AGAINST('base données MySQL' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;

-- Test 3: Recherche booléenne avancée
SELECT 
    niveau,
    source,
    LEFT(message, 60) as message,
    ip_address
FROM logs_myisam
WHERE MATCH(message, user_agent) AGAINST('+MySQL -Apache' IN BOOLEAN MODE);

-- Test 4: Recherche avec wildcard
SELECT 
    niveau,
    source,
    message
FROM logs_myisam
WHERE MATCH(message) AGAINST('sauvegard*' IN BOOLEAN MODE);

-- Test 5: Recherche combinée avec conditions normales
SELECT 
    niveau,
    COUNT(*) as nb_occurrences,
    GROUP_CONCAT(DISTINCT source) as sources
FROM logs_myisam
WHERE MATCH(message) AGAINST('erreur connexion échec' IN NATURAL LANGUAGE MODE)
  AND niveau IN ('ERROR', 'WARNING')
GROUP BY niveau;
```

**Étape 4 : Test des limitations MyISAM**
```sql
-- Test: MyISAM ne supporte PAS les transactions
START TRANSACTION;

INSERT INTO logs_myisam (niveau, source, message, ip_address) 
VALUES ('INFO', 'tp_formation', 'Test de transaction MyISAM - cette ligne devrait rester', '127.0.0.1');

-- Vérifier que la ligne est immédiatement visible (pas de transaction)
SELECT * FROM logs_myisam WHERE source = 'tp_formation';

-- Même avec ROLLBACK, l'insertion MyISAM est persistante
ROLLBACK;

-- Vérifier que la ligne est toujours là
SELECT COUNT(*) as lignes_test FROM logs_myisam WHERE source = 'tp_formation';
-- Résultat attendu : 1 (la ligne reste malgré le ROLLBACK)

-- Nettoyer pour la suite
DELETE FROM logs_myisam WHERE source = 'tp_formation';

-- Démonstration du verrouillage au niveau table
-- (Plus difficile à montrer en solo, mais MyISAM verrouille toute la table lors d'écritures)
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
    data VARCHAR(1000),
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
