
# 🔐 Connexion SSH sans mot de passe à un VPS (herge.me)

## 1. Générer une clé SSH

```bash
ssh-keygen -t ed25519 -C "connexion-herge" -f ~/.ssh/herge_key -N ""
```

- Fichiers générés :
  - Clé privée : `~/.ssh/herge_key`
  - Clé publique : `~/.ssh/herge_key.pub`

---

## 2. Copier la clé publique sur le VPS

```bash
ssh-copy-id -i ~/.ssh/herge_key.pub hgcode@herge.me
```

> Si `ssh-copy-id` est indisponible :

```bash
cat ~/.ssh/herge_key.pub | ssh hgcode@herge.me 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh'
```

---

## 3. Créer un alias SSH (optionnel mais pratique)

Dans `~/.ssh/config` :

```ssh
Host herge
  HostName herge.me
  User hgcode
  IdentityFile ~/.ssh/herge_key
```

---

## 4. Connexion simple

```bash
ssh herge
```

---

## 5. (Optionnel) Désactiver l’authentification par mot de passe sur le VPS

Éditer le fichier de configuration SSH :

```bash
sudo nano /etc/ssh/sshd_config
```

Vérifier les lignes suivantes :

```ini
PasswordAuthentication no
PubkeyAuthentication yes
```

Redémarrer SSH :

```bash
sudo systemctl restart ssh
```

---

## 🧳 Si tu changes de machine locale

Tu dois copier la clé privée `herge_key` vers la nouvelle machine, puis :

```bash
chmod 600 ~/.ssh/herge_key
chmod 644 ~/.ssh/herge_key.pub
```

Et recréer le fichier `~/.ssh/config`.

---

**⚠️ Ne partage jamais ta clé privée.**
