# 🛠️ Procédure d'installation d'un serveur web Apache + MySQL + phpMyAdmin

Cette procédure couvre :

- Apache + UFW + Fail2Ban  
- MySQL sécurisé  
- PhpMyAdmin sur un sous-domaine  
- Hiérarchie `/public` et `/log` par sous-domaine  
- Protection `.htaccess`  
- Structure de configuration virtuelle organisée par incréments de 100

---

## 🔧 1. Installer Apache et mettre à jour le pare-feu

### Mettre à jour les paquets
```bash
sudo apt update
```

### Installer Apache
```bash
sudo apt install -y apache2
```

### Autoriser Apache dans UFW
```bash
sudo ufw app list
```

**Sortie attendue :**
```
Available applications:
  Apache
  Apache Full
  Apache Secure
  OpenSSH
```

Autoriser seulement le trafic HTTP (port 80) :
```bash
sudo ufw allow in "Apache"
```

### Vérifier l’état d’UFW
```bash
sudo ufw status
```

### Tester Apache
Accéder à `http://[IP_DU_SERVEUR]` depuis un navigateur.

---

## 🌐 2. Créer un sous-domaine personnalisé

### Exemple : `phpmyadmin.tondomaine.com`

### Créer la hiérarchie de dossiers
```bash
sudo mkdir -p /var/www/phpmyadmin.tondomaine.com/public
sudo mkdir -p /var/www/phpmyadmin.tondomaine.com/log
```

### Attribuer les permissions
```bash
sudo chown -R www-data:www-data /var/www/phpmyadmin.tondomaine.com
sudo chmod -R 755 /var/www/phpmyadmin.tondomaine.com
```

### Créer un fichier Apache pour le sous-domaine
```bash
sudo nano /etc/apache2/sites-available/100-phpmyadmin.conf
```

**Contenu du fichier :**
```apache
<VirtualHost *:80>
    ServerName phpmyadmin.tondomaine.com

    DocumentRoot /var/www/phpmyadmin.tondomaine.com/public
    ErrorLog /var/www/phpmyadmin.tondomaine.com/log/error.log
    CustomLog /var/www/phpmyadmin.tondomaine.com/log/access.log combined

    <Directory /var/www/phpmyadmin.tondomaine.com/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### Activer le site et recharger Apache
```bash
sudo a2ensite 100-phpmyadmin.conf
sudo systemctl reload apache2
```

### Ajouter l’entrée DNS
Sur votre gestionnaire DNS, ajoutez :
- Type : A
- Nom : `phpmyadmin`
- Valeur : adresse IP de votre serveur

---

## 🐘 3. Installer PHP et PhpMyAdmin

### Installer PHP
```bash
sudo apt install -y php libapache2-mod-php php-mysql
```

### Installer phpMyAdmin
```bash
sudo apt install -y phpmyadmin
```

- Sélectionner `apache2` comme serveur web
- Choisir `dbconfig-common`
- Définir un mot de passe

### Lier phpMyAdmin au sous-domaine
```bash
sudo ln -s /usr/share/phpmyadmin /var/www/phpmyadmin.tondomaine.com/public
```

---

## 🗃️ 4. Installer MySQL et sécuriser l’accès

### Installer MySQL Server
```bash
sudo apt install -y mysql-server
```

### Modifier le mode d’authentification de `root`
```bash
sudo mysql
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'mot_de_passe';
exit;
```

### Lancer la configuration sécurisée
```bash
sudo mysql_secure_installation
```

Répondre aux questions :
1. Activer VALIDATE PASSWORD PLUGIN : y/n
2. Choisir un niveau de sécurité : 0 (LOW), 1 (MEDIUM), 2 (STRONG)
3. Définir le mot de passe root
4. Confirmer l’utilisation du mot de passe
5. Supprimer les utilisateurs anonymes : y
6. Interdire le login root à distance : y
7. Supprimer la base `test` : y
8. Recharger les tables de privilèges : y

### Tester la connexion
```bash
sudo mysql -u root -p
```

---

## 🔐 5. Restreindre l’accès avec `.htaccess`

### Créer le fichier `.htpasswd`
```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/apache2/.htpasswd utilisateur
```

### Créer le fichier `.htaccess`
```bash
nano /var/www/phpmyadmin.tondomaine.com/public/.htaccess
```

**Contenu :**
```apache
AuthType Basic
AuthName "Accès restreint"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user
```

Assurez-vous que `AllowOverride All` est bien activé dans la configuration Apache.

---

## 🔒 6. Sécuriser le serveur avec UFW & Fail2Ban

### Autoriser SSH et Apache
```bash
sudo ufw allow OpenSSH
sudo ufw allow "Apache"
sudo ufw enable
sudo ufw status
```

### Installer Fail2Ban
```bash
sudo apt install -y fail2ban
```

### Configurer Fail2Ban
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

**Exemple pour SSH :**
```ini
[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
maxretry = 5
```

### Redémarrer Fail2Ban
```bash
sudo systemctl restart fail2ban
```

---

## ✅ 7. Activer les modules Apache nécessaires

```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo systemctl restart apache2
```

---

## 📎 8. (Optionnel) Ajouter un certificat HTTPS avec Let’s Encrypt

### Installer Certbot
```bash
sudo apt install -y certbot python3-certbot-apache
```

### Générer le certificat SSL
```bash
sudo certbot --apache -d phpmyadmin.tondomaine.com
```

### Tester le renouvellement automatique
```bash
sudo certbot renew --dry-run
```

---

## 🎉 Serveur prêt

Votre serveur web est maintenant opérationnel avec :

- Apache, PHP et MySQL fonctionnels  
- phpMyAdmin sécurisé sur un sous-domaine  
- Arborescence claire : `/public` + `/log`  
- Pare-feu actif (UFW)  
- Anti-brute-force (Fail2Ban)  
- `.htaccess` pour les zones sensibles  
