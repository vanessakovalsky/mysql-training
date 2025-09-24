# TP 1 : Installation Complète MySQL (45 minutes)

#### 🎯 Objectifs
- Installer MySQL Server et les outils clients
- Configurer la sécurité de base
- Effectuer les premiers tests de connectivité
- Personnaliser la configuration initiale

---

#### 📋 **Partie A : Installation sur Linux Ubuntu/Debian** (20 min)

**Étape 1 : Préparation du système**
```bash
# Mettre à jour la liste des paquets
sudo apt update

# Vérifier si MySQL est déjà installé
mysql --version
# Si une version est déjà présente, notez-la

# Vérifier l'espace disque disponible (minimum 1GB recommandé)
df -h
```

**Étape 2 : Installation du serveur MySQL**
```bash
# Installer MySQL Server et client
sudo apt install mysql-server mysql-client

# ✅ Vérification : Le service doit démarrer automatiquement
sudo systemctl status mysql
# Vous devez voir "active (running)"
```

**Étape 3 : Configuration sécurisée**
```bash
# Lancer l'assistant de sécurisation
sudo mysql_secure_installation
```

**Répondez aux questions comme suit :**
- `Validate password component?` → **Y** (Oui)
- `Password validation policy` → **1** (MEDIUM)
- `New password for root` → **FormationMySQL2024!**   (optionnel il est possible selon votre OS qu'il n'y ait pas de mot de passe à définir pour root)
- `Re-enter new password` → **FormationMySQL2024!**  (optionnel il est possible selon votre OS qu'il n'y ait pas de mot de passe à définir pour root)
- `Remove anonymous users?` → **Y** (Oui)
- `Disallow root login remotely?` → **N** (Non, pour la formation)
- `Remove test database?` → **Y** (Oui)
- `Reload privilege tables?` → **Y** (Oui)

**Étape 4 : Premier test de connexion**
```bash
# Si vous avez défini un mot de passe root Se connecter en tant que root
mysql -u root -p
# Saisir le mot de passe : FormationMySQL2024!
# Dans le cas contraire
sudo mysql -u root

# Une fois connecté, exécuter :
```
```sql
-- Vérifier la version
SELECT VERSION();

-- Voir les bases de données
SHOW DATABASES;

-- Quitter MySQL
EXIT;
```

**✅ Point de contrôle :** Vous devez voir la version de MySQL et les bases système.

---

#### 📋 **Partie B : Installation sur Windows** (20 min)

**Étape 1 : Téléchargement**
1. Aller sur https://dev.mysql.com/downloads/installer/
2. Télécharger **MySQL Installer** (version complète ~400MB)
3. Double-cliquer sur le fichier téléchargé

**Étape 2 : Installation guidée**
1. **Setup Type** → Choisir **"Custom"**
2. **Select Products** → Cocher :
   - MySQL Server 8.0.x (latest)
   - MySQL Workbench 8.0.x
   - MySQL Shell 8.0.x
3. **Installation** → Cliquer **"Execute"**

**Étape 3 : Configuration du serveur**
1. **High Availability** → **"Standalone MySQL Server"**
2. **Type and Networking** :
   - Config Type : **"Development Computer"**
   - Port : **3306** (défaut)
   - ☑️ **"Open Windows Firewall"**
3. **Authentication Method** : **"Use Strong Password Encryption"**
4. **Accounts and Roles** :
   - Root Password : **FormationMySQL2024!**
   - Créer un utilisateur : **admin** / **FormationMySQL2024!**

**Étape 4 : Test avec MySQL Workbench**
1. Ouvrir **MySQL Workbench**
2. Cliquer sur la connexion **"Local instance MySQL80"**
3. Saisir le mot de passe root
4. Dans l'onglet Query, exécuter :
```sql
SELECT VERSION();
SHOW DATABASES;
```

**✅ Point de contrôle :** Workbench affiche les résultats des requêtes.

---

#### 📋 **Partie C : Configuration post-installation** (15 min)

**Étape 1 : Création d'un utilisateur de formation**
```sql
-- Se connecter en tant que root
mysql -u root -p

-- Créer un utilisateur pour la formation
CREATE USER 'formation'@'%' IDENTIFIED BY 'FormationTP2024!';

-- Donner tous les privilèges (attention : uniquement en formation)
GRANT ALL PRIVILEGES ON *.* TO 'formation'@'%' WITH GRANT OPTION;

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Vérifier les utilisateurs créés
SELECT User, Host FROM mysql.user WHERE User IN ('root', 'formation');
```

**Étape 2 : Test de l'utilisateur formation**
```bash
# Nouvelle connexion avec l'utilisateur formation
mysql -u formation -p
# Mot de passe : FormationTP2024!
```

**Étape 3 : Tests fonctionnels**
```sql
-- Créer une base de test
CREATE DATABASE test_installation;
USE test_installation;

-- Créer une table de test
CREATE TABLE test_table (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(50),
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insérer des données test
INSERT INTO test_table (nom) VALUES 
('Test 1'), 
('Test 2'), 
('Installation OK');

-- Vérifier les données
SELECT * FROM test_table;

-- Nettoyer
DROP DATABASE test_installation;
```

**✅ Résultat attendu :** 3 lignes avec id, nom et timestamp.

## Validation TP1 - Installation
- [ ] MySQL Server installé et fonctionnel
- [ ] Utilisateur 'formation' créé avec les bons privilèges
- [ ] Connexion possible via ligne de commande et outil graphique
- [ ] Base de test créée et supprimée sans erreur

## Troubleshooting

### Si mysqlworkbench n'est pas installé

* Télécharger le binaire correspondant à votre système ici : https://dev.mysql.com/downloads/workbench/
* Lancer l'installation

### Mot de passe root perdu
## Step 1: Stop MySQL completely

Run the proper stop command for your OS:

**Linux (systemd)**

```bash
sudo systemctl stop mysql
# or
sudo systemctl stop mysqld
```

**Linux (SysVinit)**

```bash
sudo service mysql stop
```

**Windows**
Stop the MySQL service in `services.msc`.

---

## Step 2: Confirm MySQL has stopped

```bash
ps aux | grep mysqld
```

* You should **not see** the main `mysqld` process.
* If it’s still running, kill it manually:

```bash
sudo kill -9 <PID>
```

---

## Step 3: Start MySQL without password check

```bash
sudo mysqld_safe --skip-grant-tables &
```

---

## Step 4: Reset root password

```bash
mysql -u root
```

Then inside MySQL shell:

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPassword!';
FLUSH PRIVILEGES;
EXIT;
```

---

## Step 5: Restart MySQL normally

```bash
sudo systemctl stop mysql
sudo systemctl start mysql
```

---

💡 **Tip:** If you don’t want to stop MySQL, you can also reset the root password using **`ALTER USER` directly** if you have another user with sufficient privileges.
