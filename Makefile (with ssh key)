.PHONY: build deploy start stop restart rollback clean_backups

USER = xxxxx
IP = xxx.xxx.xxx.xxx
REMOTE_DIR = ~/remote-folder
APP_NAME = xxxxx
BACKUP_DIR = ~/backups/xxxxx
# Usage : make clean_backups KEEP=5
KEEP = 3

# Local build
build:
	npm install
	npm run build

# Déployer avec backup auto
deploy:
	ssh $(USER)@$(IP) "\
		mkdir -p $(BACKUP_DIR) && \
		if [ -d $(REMOTE_DIR) ]; then \
			tar -czf $(BACKUP_DIR)/herge_`date +%Y%m%d-%H%M%S`.tar.gz -C $(REMOTE_DIR)/ .; \
		fi"
	rsync -avz \
		--exclude '.next' \
		--exclude 'node_modules' \
		--exclude '.git' \
		--exclude '.DS_Store' \
		--exclude 'Icon?' \
		--exclude '._*' \
		--no-links \
		./ $(USER)@$(IP):$(REMOTE_DIR)/
	ssh $(USER)@$(IP) "\
		source ~/.nvm/nvm.sh && \
		cd $(REMOTE_DIR) && \
		npm install && \
		npm run build && \
		pm2 reload $(APP_NAME)"

# Démarrer l'app
start:
	ssh $(USER)@$(IP) "source ~/.nvm/nvm.sh && cd $(REMOTE_DIR) && pm2 start npm --name $(APP_NAME) -- start"

# Arrêter l'app
stop:
	ssh $(USER)@$(IP) "pm2 stop $(APP_NAME)"

# Redémarrer l'app
restart:
	ssh $(USER)@$(IP) "pm2 restart $(APP_NAME)"

# Rollback vers le dernier backup
rollback:
	ssh $(USER)@$(IP) '\
		latest_backup=$$(ls -t $(BACKUP_DIR)/herge_*.tar.gz | head -n 1); \
		if [ -f "$$latest_backup" ]; then \
			mkdir -p $(REMOTE_DIR) && \
			rm -rf $(REMOTE_DIR)/* && \
			tar -xzf "$$latest_backup" -C $(REMOTE_DIR) && \
			cd $(REMOTE_DIR) && \
			source ~/.nvm/nvm.sh && \
			npm install && \
			npm run build && \
			pm2 restart $(APP_NAME); \
		else \
			echo "⚠️ Aucun backup trouvé dans $(BACKUP_DIR)"; \
		fi'


# Nettoyage des anciens backups (ne garde que les X derniers)
clean_backups:
	ssh $(USER)@$(IP) "\
		cd $(BACKUP_DIR) && \
		ls -1t herge_*.tar.gz | tail -n +$(shell echo $$(($(KEEP)+1))) | xargs -r rm --"

# TO DO : envoie un email ou une notification (type Discord, Slack, etc.) après déploiement ou rollback ?

# make deploy USER=debian IP=1.2.3.4 REMOTE_DIR ?= ~/app/herge.me APP_NAME ?= monapp BACKUP_DIR ?= ~/backups/herge.me

# make deploy      # Sauvegarde automatique + déploiement
# make rollback    # Restaurer la dernière version
# make start       # Démarrer
# make stop        # Stopper
# make restart     # Redémarrer
# make clean_backups         # Garde les 3 plus récents par défaut
# make clean_backups KEEP=5  # Garde les 5 plus récents
