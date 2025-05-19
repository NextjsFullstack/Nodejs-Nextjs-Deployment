# Déploiement d'une application Node.js / Next.js sur un serveur Debian / Ubuntu (Caddy, PM2, MariaDB)

Ce guide a pour but de vous accompagner dans le déploiement d’une application **Node.js (Next.js)** sur un serveur dédié **Debian/Ubuntu**, en partant d’un serveur vierge jusqu’à un site accessible en HTTPS, sécurisé et prêt à accueillir des utilisateurs.

---

## 🔧 Plan du déploiement

1. Configuration de base du serveur
2. Installation de Node.js (Nextjs), PM2 & déploiement de l’application
3. Configuration du serveur web avec Caddy
4. Sécurisation du serveur avec un pare-feu UFW
5. Installation et configuration de la base de données (MariaDB)
6. Mise en place des Logs & Monitoring

---

## 1. Configuration de base du serveur

### 🗝️ Création de la clé SSH pour le user root (depuis votre machine locale)

```bash
ssh-keygen -t ed25519 -C "votre-email@example.com"
# Par défaut, cela crée ~/.ssh/id_ed25519 et ~/.ssh/id_ed25519.pub
```

Envoyez ensuite la clé publique au serveur :

```bash
ssh-copy-id root@IP_DU_SERVEUR
```

Ou manuellement :

```bash
cat ~/.ssh/id_ed25519.pub | ssh root@IP_DU_SERVEUR "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

### 🔄 Connexion et mise à jour initiale

```bash
ssh root@IP_DU_SERVEUR

# Mise à jour
apt update && apt upgrade -y
apt install rsync curl sudo -y
```

---

### 👤 Création d’un utilisateur non-root

```bash
adduser monuser
usermod -aG sudo monuser

# Copie des clés SSH du root vers le nouvel utilisateur
cp -R ~/.ssh /home/monuser/.ssh
chown -R monuser:monuser /home/monuser/.ssh
```

---

### 🔐 Sécurisation de SSH et limiter la connexion du user root

Modifier `/etc/ssh/sshd_config` :

```ini
PermitRootLogin no
PasswordAuthentication no
```

Puis redémarrer SSH :

```bash
systemctl restart sshd
```

Testez ensuite la connexion avec :

```bash
ssh monuser@IP_DU_SERVEUR
```

---

## 2. Installation de Node.js, PM2 & Déploiement de l’application

### 📦 Installation de Node.js via NVM

```bash
# Depuis l'utilisateur monuser
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Ajoutez dans ~/.bashrc ou ~/.zshrc
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

source ~/.bashrc

nvm install --lts
```

---

### 🧠 Installation de PM2

```bash
npm install -g pm2
pm2 startup systemd -u monuser --hp /home/monuser
```

---

### 📁 Déploiement de l’application Next.js

#### Côté local :

```bash
npm install
npm run build # Si applicable

rsync -avz \
  --exclude '.next' \
  --exclude 'node_modules' \
  --exclude '.git' \
  --exclude '.DS_Store' \
  --exclude 'Icon?' \
  --exclude '._*' \
  --no-links \
./ hgcode@IP_DU_SERVEUR:~/monapp/
```

#### Côté serveur :

```bash
cd ~/monapp
npm install
npm run build # Si applicable
```

---

### ⚙️ Configuration PM2 avec `ecosystem.config.js`

Créez ce fichier dans `/home/monuser/monapp/ecosystem.config.js` :

```js
module.exports = {
  apps: [
    {
      name: "monapp",
      script: "npm",
      args: "start",
      instances: "max",
      exec_mode: "cluster",
      autorestart: true,
      env: {
        PORT: 4000,
        NODE_ENV: "production",
      },
      out_file: "/var/log/monapp/out.log",
      error_file: "/var/log/monapp/error.log",
      log_date_format: "YYYY-MM-DD HH:mm:ss",
    },
  ],
};
```

Démarrez avec PM2 :

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

---

### 🤖 Automatiser les déploiements (optionnel)

Créez un `Makefile` :

```makefile
.PHONY: build deploy

build:
	npm install
	npm run build

deploy:
  npm install
  npm run build
  rsync -avz \
  --exclude '.next' \
  --exclude 'node_modules' \
  --exclude '.git' \
  --exclude '.DS_Store' \
  --exclude 'Icon?' \
  --exclude '._*' \
  --no-links \
  ./ hgcode@IP_DU_SERVEUR:~/monapp/
  ssh -i ~/.ssh/maclef USER@IP "source ~/.nvm/nvm.sh && cd monapp && npm install && npm run build && npx prisma generate --force && npx prisma db push && pm2 reload monapp"
```
Executer le fichier `Makefile` :

```
# Pour installer et builder localement
make build

# Pour déployer via rsync + build + reload distant
make deploy
```

---

## 3. Configuration du serveur web avec Caddy

### 🌐 Installation

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

---

### 📝 Configuration du Caddyfile

Fichier `/etc/caddy/Caddyfile` :

```
monapp.mondomaine.com {
  reverse_proxy localhost:3000
  encode gzip
}
```

Redémarrez Caddy :

```bash
caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
sudo systemctl restart caddy
```

Caddy gère automatiquement le SSL (Let's Encrypt).

---

## 4. Sécurisation du serveur avec UFW

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow "WWW Full"
sudo ufw allow 19999     # (Netdata)
sudo ufw allow 3001      # (Uptime Kuma)
sudo ufw enable
```

⚠️ **Attention** : Si vous mal configurez UFW, vous pourriez perdre l’accès SSH !

---

## 5. Installation et configuration de MariaDB

### 💾 Installation

```bash
#Installation du server mariadb
sudo apt install mariadb-server

#On sécurise ensuite notre installation à l'aide de la commande
sudo mariadb-secure-installation
```

### 🛠️ Création de la base de données

Lancez MariaDB en tant qu’administrateur :

```bash
sudo mariadb
```

Puis exécutez :

```sql
CREATE DATABASE monapp;
CREATE USER 'monapp'@'localhost' IDENTIFIED BY 'motdepassefort';
GRANT ALL PRIVILEGES ON monapp.* TO 'monapp'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Configurez vos variables d’environnement (fichier `.env`) dans l’application :

```
DATABASE_URL="mysql://monapp:motdepassefort@localhost:3306/monapp"
```

Relancez PM2 :

```bash
pm2 reload monapp
```

pour une connexion mariadb adistance Créer un tunnel SSH

```Bash
# Dans un terminal local (Linux / macOS / Windows avec WSL ou Git Bash) :
ssh -L 3307:localhost:3306 user@serveur.example.com
#⚠️ Cela redirige le port 3307 local vers le port 3306 distant (MariaDB) via SSH.

# Étape 2 — Connecter DBeaver (ou autre client)
# Dans DBeaver :
# Créer une nouvelle connexion > MariaDB
# Hôte : localhost
# Port : 3307
# Utilisateur : monapp
# Mot de passe : le mot de passe de ton utilisateur MariaDB
# Base de données : monapp (ou mysql si tu veux accéder à tout)
# 💡 Tu te connectes comme si la base de données était locale, mais elle passe par SSH.
```

---

## 5. 🚀 Mise en place des Logs & Monitoring

## 📂 1. Logs

### 🔸 a. PM2 (Next.js)

#### 🔍 Voir les logs

```bash
pm2 logs
```

#### 🧹 Nettoyer les logs

```bash
pm2 flush
```

#### 📁 Emplacement des logs

```bash
~/.pm2/logs/
```

#### 📦 Exemple `ecosystem.config.js`

```js
module.exports = {
  apps: [
    {
      name: "next-app",
      script: "npm",
      args: "start",
      out_file: "/var/log/next-app/out.log",
      error_file: "/var/log/next-app/error.log",
      log_date_format: "YYYY-MM-DD HH:mm:ss",
    },
  ],
};
```

---

### 🔸 b. Caddy

#### 🛠️ Configuration dans `Caddyfile`

```txt
yourdomain.com {
  reverse_proxy localhost:3000

  log {
    output file /var/log/caddy/access.log
    format single_field common_log
  }
}
```

#### 📁 Emplacement :

```bash
/var/log/caddy/
```

---

### 🔸 c. UFW / SSH / Système

#### 🔥 UFW :

```bash
sudo less /var/log/ufw.log
```

#### 🔐 SSH :

```bash
sudo journalctl -u ssh
```

#### 🧠 Journal système :

```bash
sudo journalctl -xe
```

---

### 🔸 d. MariaDB

#### 📁 Fichier de log :

```bash
sudo less /var/log/mysql/error.log
```

---

## 📊 2. Monitoring

### 🔹 a. PM2 + Keymetrics (App Node.js)

#### 📦 Installer l'agent :

```bash
pm2 install pm2-server-monit

pm2 [list|ls|status]

pm2 monit
```

#### 🌐 Connexion Keymetrics :

Créer un compte sur [https://app.keymetrics.io](https://app.keymetrics.io)

Puis :

```bash
pm2 link <public_key> <secret_key>
```

---

### 🔹 b. Monitoring système

#### 🟢 `htop` :

```bash
sudo apt install htop
htop
```

#### 🟠 `glances` :

```bash
sudo apt install glances
glances
```

#### 🔵 `netdata` :

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

- Accès via navigateur :  
  `http://<VPS_IP>:19999`

---

### 🔹 c. Monitoring Uptime (surveillance)

#### 🌐 Uptime Kuma (recommandé) :

```bash
docker run -d   --name uptime-kuma   -p 3001:3001   -v uptime-kuma:/app/data   louislam/uptime-kuma
```

#### 🔗 Alternatives externes :

- [UptimeRobot](https://uptimerobot.com)
- [BetterUptime](https://betteruptime.com)

---

### 🔹 d. Alertes personnalisées

#### Exemple avec cron + mail :

```bash
*/5 * * * * pm2 describe next-app | grep errored && echo "Erreur PM2" | mail -s "App Down" root@localhost
```

> Tu peux remplacer `mail` par un webhook Telegram, Slack, ou Discord selon ton besoin.

---

- 🔁 Redémarrer les services si besoin :

```bash
sudo systemctl restart caddy
pm2 reload all
```

---

## 🧠 Bonnes pratiques

- Utiliser `logrotate` pour éviter les gros fichiers :

```bash
sudo apt install logrotate
```

- Garder un œil sur l’espace disque :

```bash
df -h
du -sh /var/log/*
```

## ✅ Conclusion

Votre application **Node.js / Next.js** est maintenant :

- Déployée sur un serveur Debian propre
- Accessible via HTTPS avec Caddy
- Sécurisée avec UFW
- Reliée à une base de données MariaDB
- Maintenue active par PM2

Pour aller plus loin :

- Utilisez **Ansible** pour automatiser ce processus
- Explorez **Docker** pour isoler les composants
