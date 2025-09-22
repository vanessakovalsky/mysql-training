# TP : Gestion des Tablespaces et Index (30 minutes)

#### 🎯 Objectifs
- Créer et gérer des tablespaces personnalisés
- Comprendre l'impact des index sur les performances
- Analyser les plans d'exécution avec EXPLAIN

---

#### 📋 **Partie A : Gestion des Tablespaces** (15 min)

**Étape 1 : Explorer les tablespaces existants**
```sql
-- Voir les tablespaces système
SELECT 
    TABLESPACE_NAME,
    ENGINE,
    TOTAL_EXTENTS,
    EXTENT_SIZE,
    INITIAL_SIZE/1024/1024 as SIZE_MB
FROM INFORMATION_SCHEMA.TABLESPACES
ORDER BY TABLESPACE_NAME;

-- Voir les fichiers de données
SELECT 
    TABLESPACE_NAME,
    FILE_NAME,
    FILE_TYPE,
    TOTAL_EXTENTS * EXTENT_SIZE / 1024 / 1024 as SIZE_MB
FROM INFORMATION_SCHEMA.FILES
WHERE FILE_TYPE = 'DATAFILE'
ORDER BY TABLESPACE_NAME;
```

**Étape 2 : Créer des tablespaces personnalisés**
```sql
-- Créer un tablespace pour les données de production
CREATE TABLESPACE formation_production 
ADD DATAFILE 'formation_prod.ibd' 
FILE_BLOCK_SIZE = 16K;

-- Créer un tablespace pour les données d'archive
CREATE TABLESPACE formation_archives 
ADD DATAFILE 'formation_arch.ibd'
FILE_BLOCK_SIZE = 16K;

-- Créer un tablespace pour les données temporaires/test
CREATE TABLESPACE formation_temp 
ADD DATAFILE 'formation_temp.ibd'
FILE_BLOCK_SIZE = 8K;

-- Vérifier la création
SELECT TABLESPACE_NAME, FILE_NAME 
FROM INFORMATION_SCHEMA.FILES 
WHERE TABLESPACE_NAME LIKE 'formation_%'
ORDER BY TABLESPACE_NAME;
```

**Étape 3 : Créer des tables dans des tablespaces spécifiques**
```sql
-- Changer de base pour ce TP
CREATE DATABASE gestion_tablespaces;
USE gestion_tablespaces;

-- Table de production dans le tablespace dédié
CREATE TABLE commandes_production (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    date_commande DATE NOT NULL,
    montant DECIMAL(10,2) NOT NULL,
    statut ENUM('NOUVELLE', 'TRAITEE', 'EXPEDIEE', 'LIVREE'),
    
    INDEX idx_client (client_id),
    INDEX idx_date (date_commande),
    INDEX idx_statut (statut)
) TABLESPACE formation_production;

-- Table d'archives dans son tablespace
CREATE TABLE commandes_archives (
    id INT NOT NULL,
    client_id INT NOT NULL,
    date_commande DATE NOT NULL,
    montant DECIMAL(10,2) NOT NULL,
    statut VARCHAR(20),
    date_archivage TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id, date_archivage),
    INDEX idx_date_arch (date_archivage)
) TABLESPACE formation_archives;

-- Table temporaire pour les tests
CREATE TABLE temp_calculs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    valeur_a DECIMAL(15,5),
    valeur_b DECIMAL(15,5),
    resultat DECIMAL(15,5),
    timestamp_calcul TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) TABLESPACE formation_temp;
```

**Étape 4 : Vérifier l'organisation**
```sql
-- Voir la répartition des tables par tablespace
SELECT 
    t.TABLE_NAME,
    t.TABLESPACE_NAME,
    t.TABLE_ROWS,
    ROUND((t.DATA_LENGTH + t.INDEX_LENGTH) / 1024 / 1024, 2) as SIZE_MB
FROM INFORMATION_SCHEMA.TABLES t
WHERE t.TABLE_SCHEMA = 'gestion_tablespaces'
ORDER BY t.TABLESPACE_NAME, t.TABLE_NAME;
```

---

#### 📋 **Partie B : Gestion avancée des Index** (15 min)

**Étape 1 : Créer des données de test pour les performances**
```sql
-- Insérer des données de test dans commandes_production
INSERT INTO commandes_production (client_id, date_commande, montant, statut) 
VALUES 
-- Semaine 1
(101, '2024-01-01', 150.50, 'LIVREE'),
(102, '2024-01-01', 75.25, 'LIVREE'),
(103, '2024-01-02', 299.99, 'EXPEDIEE'),
(101, '2024-01-02', 45.00, 'TRAITEE'),
(104, '2024-01-03', 120.75, 'NOUVELLE'),
-- Semaine 2
(102, '2024-01-08', 89.50, 'LIVREE'),
(105, '2024-01-09', 199.99, 'EXPEDIEE'),
(103, '2024-01-10', 350.25, 'TRAITEE'),
(106, '2024-01-11', 78.90, 'NOUVELLE'),
(104, '2024-01-12', 225.00, 'LIVREE'),
-- Plus de données pour les tests
(107, '2024-01-15', 167.80, 'EXPEDIEE'),
(108, '2024-01-16', 98.45, 'TRAITEE'),
(109, '2024-01-17', 445.60, 'NOUVELLE'),
(110, '2024-01-18', 67.30, 'LIVREE'),
(101, '2024-01-19', 156.75, 'EXPEDIEE');

-- Générer plus de données avec une requête
INSERT INTO commandes_production (client_id, date_commande, montant, statut)
SELECT 
    101 + (id % 20) as client_id,
    DATE_ADD('2024-01-20', INTERVAL (id % 30) DAY) as date_commande,
    ROUND(RAND() * 500 + 50, 2) as montant,
    ELT(1 + (id % 4), 'NOUVELLE', 'TRAITEE', 'EXPEDIEE', 'LIVREE') as statut
FROM commandes_production
WHERE id <= 15;  -- Doubler les données existantes

-- Vérifier les données
SELECT COUNT(*) as total_commandes FROM commandes_production;
SELECT statut, COUNT(*) as nb FROM commandes_production GROUP BY statut;
```

**Étape 2 : Analyser les performances sans index**
```sql
-- Supprimer temporairement les index (sauf PRIMARY KEY)
DROP INDEX idx_client ON commandes_production;
DROP INDEX idx_date ON commandes_production;
DROP INDEX idx_statut ON commandes_production;

-- Tester une requête sans index
EXPLAIN SELECT * FROM commandes_production 
WHERE client_id = 103 AND statut = 'LIVREE';

-- Observer le plan d'exécution (FULL TABLE SCAN)
EXPLAIN FORMAT=JSON 
SELECT client_id, COUNT(*) as nb_commandes, SUM(montant) as total
FROM commandes_production 
WHERE date_commande BETWEEN '2024-01-01' AND '2024-01-15'
GROUP BY client_id;
```

**Étape 3 : Créer des index optimisés**
```sql
-- Index composé pour requêtes fréquentes
CREATE INDEX idx_client_statut ON commandes_production(client_id, statut);

-- Index sur la date pour les recherches par période
CREATE INDEX idx_date_montant ON commandes_production(date_commande, montant);

-- Index partiel pour les commandes actives uniquement
CREATE INDEX idx_commandes_actives ON commandes_production(date_commande, client_id)
WHERE statut IN ('NOUVELLE', 'TRAITEE', 'EXPEDIEE');

-- Voir les index créés
SHOW INDEX FROM commandes_production;
```

**Étape 4 : Comparer les performances**
```sql
-- Même requête qu'avant, mais avec index
EXPLAIN SELECT * FROM commandes_production 
WHERE client_id = 103 AND statut = 'LIVREE';

-- La requête groupée avec index
EXPLAIN FORMAT=JSON 
SELECT client_id, COUNT(*) as nb_commandes, SUM(montant) as total
FROM commandes_production 
WHERE date_commande BETWEEN '2024-01-01' AND '2024-01-15'
GROUP BY client_id;

-- Test de l'index partiel
EXPLAIN SELECT * FROM commandes_production 
WHERE date_commande = '2024-01-10' 
  AND client_id = 103 
  AND statut = 'TRAITEE';
```

**Étape 5 : Analyse détaillée des index**
```sql
-- Statistiques sur l'utilisation des index
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    CARDINALITY,
    SUB_PART,
    NULLABLE,
    INDEX_TYPE
FROM INFORMATION_SCHEMA.STATISTICS 
WHERE TABLE_SCHEMA = 'gestion_tablespaces' 
  AND TABLE_NAME = 'commandes_production'
ORDER BY SEQ_IN_INDEX;

-- Analyser la table pour mettre à jour les statistiques
ANALYZE TABLE commandes_production;

-- Voir l'espace utilisé par les index
SELECT 
    TABLE_NAME,
    ROUND(DATA_LENGTH/1024/1024, 2) as DATA_MB,
    ROUND(INDEX_LENGTH/1024/1024, 2) as INDEX_MB,
    ROUND(INDEX_LENGTH/DATA_LENGTH*100, 2) as INDEX_PCT
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_SCHEMA = 'gestion_tablespaces' 
  AND TABLE_NAME = 'commandes_production';
```

## 📋 **Exercices Bonus** 

**Exercice 1 : Migration entre tablespaces**
```sql
-- Créer une table dans le tablespace par défaut
CREATE TABLE test_migration (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data VARCHAR(100)
) ENGINE=InnoDB;

-- Insérer quelques données
INSERT INTO test_migration (data) VALUES 
('Donnée 1'), ('Donnée 2'), ('Donnée 3');

-- Migrer vers un tablespace personnalisé
ALTER TABLE test_migration TABLESPACE formation_temp;

-- Vérifier la migration
SELECT TABLE_NAME, TABLESPACE_NAME 
FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME = 'test_migration';
```

**Exercice 2 : Optimisation d'index**
```sql
-- Créer une requête complexe
EXPLAIN 
SELECT 
    c.client_id,
    COUNT(*) as nb_commandes,
    AVG(c.montant) as panier_moyen,
    MAX(c.date_commande) as derniere_commande
FROM commandes_production c
WHERE c.statut IN ('TRAITEE', 'EXPEDIEE', 'LIVREE')
  AND c.date_commande >= '2024-01-01'
GROUP BY c.client_id
HAVING COUNT(*) > 1
ORDER BY panier_moyen DESC
LIMIT 10;

-- Créer un index optimisé pour cette requête
CREATE INDEX idx_optimise_rapport 
ON commandes_production(statut, date_commande, client_id, montant);

-- Re-tester la requête
EXPLAIN 
SELECT 
    c.client_id,
    COUNT(*) as nb_commandes,
    AVG(c.montant) as panier_moyen,
    MAX(c.date_commande) as derniere_commande
FROM commandes_production c
WHERE c.statut IN ('TRAITEE', 'EXPEDIEE', 'LIVREE')
  AND c.date_commande >= '2024-01-01'
GROUP BY c.client_id
HAVING COUNT(*) > 1
ORDER BY panier_moyen DESC
LIMIT 10;
```

---

## 📋 **Nettoyage et Vérifications Finales**

**Nettoyage des environnements de test :**
```sql
-- Nettoyer les bases de test (optionnel)
-- DROP DATABASE test_installation;
-- DROP DATABASE test_moteurs;
-- DROP DATABASE gestion_tablespaces;
-- DROP DATABASE banque_formation;

-- Nettoyer les tablespaces (optionnel)
-- DROP TABLESPACE formation_production;
-- DROP TABLESPACE formation_archives; 
-- DROP TABLESPACE formation_temp;

-- Voir l'état final du système
SHOW DATABASES;
SHOW VARIABLES LIKE 'version';
SELECT USER(), CONNECTION_ID();
```



## Validation TP - Tablespaces et Index
- [ ] Tablespaces personnalisés créés et utilisés
- [ ] Impact des index sur les performances observé
- [ ] Plans d'exécution analysés avec EXPLAIN
- [ ] Index composés et partiels créés et testés


