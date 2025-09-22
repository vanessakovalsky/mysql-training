# TP  : Sécurité et Gestion des Utilisateurs (50 minutes)

## 🎯 Objectifs
- Sécuriser complètement une installation MySQL
- Créer différents types d'utilisateurs avec privilèges appropriés
- Implémenter une politique de mots de passe
- Comprendre les rôles et leur gestion

## 📋 **Partie A : Sécurisation complète post-installation** (15 min)

**Étape 1 : Audit de sécurité initial**
```sql
-- Se connecter en tant qu'utilisateur privilégié
mysql -u root -p

-- Vérifier les utilisateurs existants
SELECT 
    User, 
    Host, 
    plugin,
    authentication_string,
    password_expired,
    account_locked
FROM mysql.user
ORDER BY User, Host;

-- Vérifier les bases de données accessibles
SHOW DATABASES;

-- Vérifier les utilisateurs anonymes (à supprimer)
SELECT User, Host FROM mysql.user WHERE User = '';

-- Vérifier les accès root distants (à sécuriser)
SELECT User, Host FROM mysql.user WHERE User = 'root' AND Host != 'localhost';
```

**Étape 2 : Nettoyage sécuritaire**
```sql
-- Supprimer les utilisateurs anonymes
DELETE FROM mysql.user WHERE User = '';

-- Supprimer la base de données test (si elle existe)
DROP DATABASE IF EXISTS test;

-- Supprimer les privilèges sur la base test
DELETE FROM mysql.db WHERE Db = 'test' OR Db = 'test\\_%';

-- Limiter l'accès root au localhost uniquement (optionnel en formation)
-- DELETE FROM mysql.user WHERE User = 'root' AND Host != 'localhost';

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Vérification finale
SELECT User, Host FROM mysql.user ORDER BY User, Host;
```

**Étape 3 : Configuration de la politique de mots de passe**
```sql
-- Voir la configuration actuelle
SHOW VARIABLES LIKE 'validate_password%';

-- Configurer la politique de validation (si le plugin est installé)
-- SET GLOBAL validate_password.policy = 'STRONG';
-- SET GLOBAL validate_password.length = 12;
-- SET GLOBAL validate_password.mixed_case_count = 1;
-- SET GLOBAL validate_password.number_count = 2;
-- SET GLOBAL validate_password.special_char_count = 1;

-- Configurer l'expiration automatique des mots de passe
SET GLOBAL default_password_lifetime = 90; -- 90 jours
```

## 📋 **Partie B : Création d'utilisateurs par profil** (25 min)

**Étape 1 : Utilisateur application web (accès limité)**
```sql
-- Créer l'utilisateur pour l'application web
CREATE USER 'app_web'@'192.168.%.%' 
IDENTIFIED BY 'WebApp2024#Secure!' 
PASSWORD EXPIRE INTERVAL 60 DAY;

-- Privilèges sur la base de données application
GRANT SELECT, INSERT, UPDATE ON formation_procedural.clients TO 'app_web'@'192.168.%.%';
GRANT SELECT, INSERT, UPDATE ON formation_procedural.commandes TO 'app_web'@'192.168.%.%';
GRANT SELECT ON formation_procedural.produits TO 'app_web'@'192.168.%.%';

-- Privilèges sur les procédures spécifiques
GRANT EXECUTE ON PROCEDURE formation_procedural.generer_commandes_test TO 'app_web'@'192.168.%.%';
GRANT EXECUTE ON FUNCTION formation_procedural.calculer_remise_client TO 'app_web'@'192.168.%.%';
GRANT EXECUTE ON FUNCTION formation_procedural.statut_lisible TO 'app_web'@'192.168.%.%';

-- Limiter les ressources
ALTER USER 'app_web'@'192.168.%.%'
WITH MAX_QUERIES_PER_HOUR 1000
     MAX_UPDATES_PER_HOUR 500
     MAX_CONNECTIONS_PER_HOUR 50;

-- Vérifier les privilèges accordés
SHOW GRANTS FOR 'app_web'@'192.168.%.%';
```

**Étape 2 : Utilisateur lecture seule (reporting)**
```sql
-- Créer l'utilisateur pour les rapports
CREATE USER 'reporting'@'%' 
IDENTIFIED BY 'Report2024@Reader!' 
PASSWORD EXPIRE INTERVAL 180 DAY;

-- Privilèges de lecture seule sur toutes les tables de données
GRANT SELECT ON formation_procedural.clients TO 'reporting'@'%';
GRANT SELECT ON formation_procedural.commandes TO 'reporting'@'%';
GRANT SELECT ON formation_procedural.produits TO 'reporting'@'%';
GRANT SELECT ON formation_procedural.stats_ventes_mensuelles TO 'reporting'@'%';
GRANT SELECT ON formation_procedural.audit_operations TO 'reporting'@'%';

-- Privilèges sur les fonctions de calcul
GRANT EXECUTE ON FUNCTION formation_procedural.calculer_remise_client TO 'reporting'@'%';
GRANT EXECUTE ON FUNCTION formation_procedural.statut_lisible TO 'reporting'@'%';

-- Privilèges sur les procédures de statistiques
GRANT EXECUTE ON PROCEDURE formation_procedural.calculer_stats_mensuelles TO 'reporting'@'%';

-- Pas de limitation de ressources pour cet utilisateur (rapports lourds)
SHOW GRANTS FOR 'reporting'@'%';
```

**Étape 3 : Utilisateur administrateur base de données**
```sql
-- Créer l'utilisateur DBA
CREATE USER 'dba_formation'@'localhost' 
IDENTIFIED BY 'DBA2024#FormationSecure!' 
PASSWORD EXPIRE INTERVAL 30 DAY;

-- Privilèges administrateur sur la base formation
GRANT ALL PRIVILEGES ON formation_procedural.* TO 'dba_formation'@'localhost' WITH GRANT OPTION;

-- Privilèges de maintenance globaux
GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'dba_formation'@'localhost';

-- Privilèges sur la base mysql pour gestion des utilisateurs
GRANT SELECT ON mysql.user TO 'dba_formation'@'localhost';
GRANT SELECT ON mysql.db TO 'dba_formation'@'localhost';

SHOW GRANTS FOR 'dba_formation'@'localhost';
```

**Étape 4 : Utilisateur temporaire avec limitations**
```sql
-- Créer un utilisateur temporaire (stagiaire)
CREATE USER 'stagiaire_temp'@'%' 
IDENTIFIED BY 'Temp2024#Stagiaire!' 
PASSWORD EXPIRE INTERVAL 7 DAY
ACCOUNT LOCK;

-- Privilèges très limités
GRANT SELECT ON formation_procedural.produits TO 'stagiaire_temp'@'%';
GRANT SELECT (nom, email, type_client) ON formation_procedural.clients TO 'stagiaire_temp'@'%';

-- Limitations strictes
ALTER USER 'stagiaire_temp'@'%'
WITH MAX_QUERIES_PER_HOUR 50
     MAX_UPDATES_PER_HOUR 0
     MAX_CONNECTIONS_PER_HOUR 5
     MAX_USER_CONNECTIONS 2;

-- Déverrouiller le compte quand nécessaire
ALTER USER 'stagiaire_temp'@'%' ACCOUNT UNLOCK;

SHOW GRANTS FOR 'stagiaire_temp'@'%';
```

## 📋 **Partie C : Gestion des rôles (MySQL 8.0+)** (10 min)

**Étape 1 : Créer des rôles par fonction**
```sql
-- Créer les rôles de base
CREATE ROLE 'role_lecteur', 'role_editeur', 'role_analyste', 'role_admin';

-- Définir les privilèges du rôle lecteur
GRANT SELECT ON formation_procedural.* TO 'role_lecteur';
GRANT EXECUTE ON FUNCTION formation_procedural.statut_lisible TO 'role_lecteur';

-- Définir les privilèges du rôle éditeur (inclut lecteur)
GRANT 'role_lecteur' TO 'role_editeur';
GRANT INSERT, UPDATE ON formation_procedural.clients TO 'role_editeur';
GRANT INSERT, UPDATE ON formation_procedural.commandes TO 'role_editeur';

-- Définir les privilèges du rôle analyste (inclut lecteur + procédures)
GRANT 'role_lecteur' TO 'role_analyste';
GRANT EXECUTE ON PROCEDURE formation_procedural.calculer_stats_mensuelles TO 'role_analyste';
GRANT SELECT, INSERT, UPDATE ON formation_procedural.stats_ventes_mensuelles TO 'role_analyste';

-- Définir les privilèges du rôle admin (tous les privilèges)
GRANT ALL PRIVILEGES ON formation_procedural.* TO 'role_admin' WITH GRANT OPTION;

-- Voir les rôles créés
SELECT * FROM mysql.user WHERE User LIKE 'role_%';
```

**Étape 2 : Attribuer les rôles aux utilisateurs**
```sql
-- Créer de nouveaux utilisateurs avec rôles
CREATE USER 'user_lecteur'@'%' IDENTIFIED BY 'Lecteur2024#';
CREATE USER 'user_editeur'@'%' IDENTIFIED BY 'Editeur2024#';
CREATE USER 'user_analyste'@'%' IDENTIFIED BY 'Analyste2024#';

-- Attribuer les rôles
GRANT 'role_lecteur' TO 'user_lecteur'@'%';
GRANT 'role_editeur' TO 'user_editeur'@'%';
GRANT 'role_analyste' TO 'user_analyste'@'%';

-- Définir les rôles par défaut
SET DEFAULT ROLE 'role_lecteur' TO 'user_lecteur'@'%';
SET DEFAULT ROLE 'role_editeur' TO 'user_editeur'@'%';
SET DEFAULT ROLE 'role_analyste' TO 'user_analyste'@'%';

-- Vérifier les attributions
SHOW GRANTS FOR 'user_editeur'@'%';
SHOW GRANTS FOR 'user_analyste'@'%';
```

---

#### 📋 **Tests et Validation** (bonus)

**Test 1 : Connexion avec utilisateur limité**
```bash
# Tester la connexion avec l'utilisateur app_web
mysql -u app_web -h localhost -p

# Une fois connecté, tester les privilèges
```

```sql
-- Doit fonctionner
SELECT * FROM formation_procedural.produits LIMIT 5;

-- Doit fonctionner
SELECT calculer_remise_client(150.00, 'PARTICULIER') as remise;

-- Doit échouer (pas de privilège DROP)
DROP TABLE formation_procedural.produits;

-- Doit échouer (pas de privilège sur audit_operations)
SELECT * FROM formation_procedural.audit_operations;
```

**Test 2 : Validation des limitations de ressources**
```sql
-- Voir les limites actuelles d'un utilisateur
SELECT 
    User, 
    Host, 
    max_questions, 
    max_updates, 
    max_connections, 
    max_user_connections
FROM mysql.user 
WHERE User = 'app_web';

-- Voir les connexions actuelles par utilisateur
SELECT 
    USER, 
    HOST, 
    COUNT(*) as nb_connexions
FROM INFORMATION_SCHEMA.PROCESSLIST 
GROUP BY USER, HOST;
```

## ✅ **Sécurité et Administration - Points de Contrôle**

**Nettoyage Post-Installation :**
- [ ] Utilisateurs anonymes supprimés
- [ ] Base de données test supprimée
- [ ] Accès root sécurisé
- [ ] Politique de mots de passe configurée

**Gestion des Utilisateurs :**
- [ ] Utilisateur application avec privilèges limités
- [ ] Utilisateur lecture seule fonctionnel
- [ ] Utilisateur DBA avec privilèges appropriés
- [ ] Limitations de ressources appliquées

**Rôles (MySQL 8.0+) :**
- [ ] Rôles créés par fonction
- [ ] Privilèges attribués aux rôles
- [ ] Rôles assignés aux utilisateurs
- [ ] Rôles par défaut configurés

### ✅ **Tests de Validation**

**Tests Fonctionnels :**
- [ ] Connexion avec chaque type d'utilisateur réussie
- [ ] Privilèges respectés (accès autorisé/refusé)
- [ ] Procédures stockées exécutables selon les droits
- [ ] Triggers déclenché automatiquement
- [ ] Audit des opérations fonctionnel

**Tests de Sécurité :**
- [ ] Tentative d'accès non autorisé bloquée
- [ ] Limitations de ressources effectives
- [ ] Mots de passe complexes imposés
- [ ] Sessions utilisateurs gérées correctement
