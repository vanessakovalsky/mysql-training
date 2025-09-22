# TP  : Mise en Œuvre de Transactions (40 minutes)

## 🎯 Objectifs
- Créer un système bancaire simple
- Comprendre les transactions ACID
- Tester les niveaux d'isolation
- Gérer les cas d'erreur avec ROLLBACK


## 📋 **Partie A : Création de la base de données bancaire** (10 min)

**Étape 1 : Créer l'environnement**
```sql
-- Se connecter
mysql -u formation -p

-- Créer la base de données
CREATE DATABASE banque_formation;
USE banque_formation;

-- Vérifier qu'on est dans la bonne base
SELECT DATABASE();
```

**Étape 2 : Créer les tables**
```sql
-- Table des comptes
CREATE TABLE comptes (
    numero VARCHAR(20) PRIMARY KEY,
    titulaire VARCHAR(100) NOT NULL,
    solde DECIMAL(12,2) DEFAULT 0,
    date_ouverture TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    actif BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- Table des opérations
CREATE TABLE operations (
    id INT PRIMARY KEY AUTO_INCREMENT,
    compte_source VARCHAR(20),
    compte_dest VARCHAR(20),
    montant DECIMAL(12,2) NOT NULL,
    type ENUM('DEPOT', 'RETRAIT', 'VIREMENT') NOT NULL,
    date_operation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    description TEXT,
    statut ENUM('PENDING', 'COMPLETED', 'CANCELLED') DEFAULT 'PENDING',
    
    FOREIGN KEY (compte_source) REFERENCES comptes(numero),
    FOREIGN KEY (compte_dest) REFERENCES comptes(numero)
) ENGINE=InnoDB;

-- Vérifier la création
SHOW TABLES;
DESCRIBE comptes;
DESCRIBE operations;
```

**Étape 3 : Insérer les données initiales**
```sql
-- Créer des comptes de test
INSERT INTO comptes (numero, titulaire, solde) VALUES
('FR001234567890', 'Jean Dupont', 1500.00),
('FR009876543210', 'Marie Martin', 2300.00),
('FR001111111111', 'Pierre Durand', 800.00),
('FR002222222222', 'Sophie Dubois', 150.00);

-- Vérifier les données
SELECT numero, titulaire, solde FROM comptes ORDER BY solde DESC;
```

**✅ Point de contrôle :** 4 comptes créés avec leurs soldes initiaux.

---

## 📋 **Partie B : Transactions simples** (15 min)

**Étape 1 : Dépôt simple (transaction automatique)**
```sql
-- Déposer 500€ sur le compte de Jean Dupont
UPDATE comptes SET solde = solde + 500 
WHERE numero = 'FR001234567890';

-- Enregistrer l'opération
INSERT INTO operations (compte_dest, montant, type, description, statut)
VALUES ('FR001234567890', 500.00, 'DEPOT', 'Dépôt formation TP', 'COMPLETED');

-- Vérifier le nouveau solde
SELECT numero, titulaire, solde FROM comptes 
WHERE numero = 'FR001234567890';
```

**Étape 2 : Retrait avec vérification (transaction manuelle)**
```sql
-- Démarrer une transaction explicite
START TRANSACTION;

-- Vérifier le solde avant retrait
SELECT solde FROM comptes WHERE numero = 'FR001234567890';

-- Supposons un retrait de 300€
UPDATE comptes SET solde = solde - 300 
WHERE numero = 'FR001234567890' AND solde >= 300;

-- Vérifier que la mise à jour a eu lieu (ROW_COUNT > 0)
SELECT ROW_COUNT() as lignes_modifiees;

-- Si ROW_COUNT() = 1, c'est OK, sinon il faut annuler
-- Pour ce TP, on suppose que c'est OK

-- Enregistrer l'opération
INSERT INTO operations (compte_source, montant, type, description, statut)
VALUES ('FR001234567890', 300.00, 'RETRAIT', 'Retrait formation TP', 'COMPLETED');

-- Valider la transaction
COMMIT;

-- Vérifier le résultat
SELECT numero, solde FROM comptes WHERE numero = 'FR001234567890';
```

**✅ Résultat attendu :** Solde = 1500 + 500 - 300 = 1700€


## 📋 **Partie C : Virement complexe avec gestion d'erreur** (15 min)

**Étape 1 : Virement réussi**
```sql
-- Virement de 200€ de Jean (FR001234567890) vers Marie (FR009876543210)
START TRANSACTION;

-- Étape 1 : Vérifier le solde du compte source
SELECT numero, titulaire, solde FROM comptes 
WHERE numero = 'FR001234567890';

-- Étape 2 : Débiter le compte source (vérifier solde suffisant)
UPDATE comptes SET solde = solde - 200 
WHERE numero = 'FR001234567890' AND solde >= 200;

-- Vérifier que le débit a fonctionné
SELECT ROW_COUNT() as debit_ok;

-- Si ROW_COUNT() = 0, il faut faire ROLLBACK
-- Pour ce TP, continuons (solde suffisant)

-- Étape 3 : Créditer le compte destination
UPDATE comptes SET solde = solde + 200 
WHERE numero = 'FR009876543210';

-- Étape 4 : Enregistrer l'opération
INSERT INTO operations (compte_source, compte_dest, montant, type, description, statut)
VALUES ('FR001234567890', 'FR009876543210', 200.00, 'VIREMENT', 
        'Virement formation - Transaction réussie', 'COMPLETED');

-- Étape 5 : Vérifier les soldes avant validation
SELECT numero, titulaire, solde FROM comptes 
WHERE numero IN ('FR001234567890', 'FR009876543210')
ORDER BY numero;

-- Étape 6 : Tout est OK, valider
COMMIT;

-- Vérification finale
SELECT numero, titulaire, solde FROM comptes 
WHERE numero IN ('FR001234567890', 'FR009876543210')
ORDER BY numero;
```

**Étape 2 : Virement avec échec (solde insuffisant)**
```sql
-- Tentative de virement de 5000€ de Sophie (150€ de solde) vers Pierre
START TRANSACTION;

-- Vérifier le solde source
SELECT numero, titulaire, solde FROM comptes 
WHERE numero = 'FR002222222222';

-- Tenter le débit (va échouer)
UPDATE comptes SET solde = solde - 5000 
WHERE numero = 'FR002222222222' AND solde >= 5000;

-- Vérifier le résultat
SELECT ROW_COUNT() as debit_reussi;

-- ROW_COUNT() = 0 car solde insuffisant
-- Annuler la transaction
ROLLBACK;

-- Enregistrer l'opération échouée
INSERT INTO operations (compte_source, compte_dest, montant, type, description, statut)
VALUES ('FR002222222222', 'FR001111111111', 5000.00, 'VIREMENT', 
        'Virement formation - ECHEC solde insuffisant', 'CANCELLED');

-- Vérifier que les soldes n'ont pas changé
SELECT numero, titulaire, solde FROM comptes 
WHERE numero IN ('FR002222222222', 'FR001111111111')
ORDER BY numero;
```

**✅ Point de contrôle :** Les soldes de Sophie et Pierre sont inchangés, mais l'opération est enregistrée avec le statut CANCELLED.


## Validation TP - Transactions
- [ ] Base banque_formation créée avec contraintes d'intégrité
- [ ] Transaction simple (dépôt) réalisée avec succès
- [ ] Virement complexe avec COMMIT/ROLLBACK maîtrisé
- [ ] Gestion des erreurs (solde insuffisant) implémentée
- [ ] Différence entre InnoDB et MyISAM comprise

