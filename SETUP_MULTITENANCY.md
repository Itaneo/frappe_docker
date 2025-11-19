# Configuration ERPNext Multitenant - Documentation

## Vue d'ensemble

Configuration d'un environnement ERPNext multitenant avec deux sites :
- **erp.hexalith.com** : Site Itaneo (ERPNext standard)
- **vente.hexalith.com** : Site Real Estate (ERPNext + app erpnext-real-estate-sales)

## Architecture

- **Nginx externe** : Gère HTTPS et reverse proxy (configuré via install-nginx-hexalith.sh)
- **ERPNext Docker** : Services backend, frontend, websocket, workers
- **MariaDB 11.8** : Base de données
- **Redis** : Cache et queue
- **Port exposé** : 8080 (pour Nginx externe)

---

## Étape 1 : Configuration de l'environnement (.env)

### Fichier créé : `/home/quentindv/frappe_docker/.env`

```bash
# Version ERPNext
ERPNEXT_VERSION=v15.89.0

# Mot de passe base de données
DB_PASSWORD=Protege est le mot de passe de la db

# Configuration multitenancy
FRAPPE_SITE_NAME_HEADER=$$host  # Routage par domaine

# Port exposé pour Nginx externe
HTTP_PUBLISH_PORT=8080

# Configuration proxy
UPSTREAM_REAL_IP_ADDRESS=127.0.0.1
UPSTREAM_REAL_IP_HEADER=X-Forwarded-For
UPSTREAM_REAL_IP_RECURSIVE=off

# Timeouts et limites
PROXY_READ_TIMEOUT=120s
CLIENT_MAX_BODY_SIZE=50m

# Liste des sites (pour référence)
SITES=`erp.hexalith.com`,`vente.hexalith.com`
```

### Points importants :
- `FRAPPE_SITE_NAME_HEADER=$$host` : Active le routage par nom de domaine
- Le mot de passe DB doit correspondre à celui utilisé lors de la création des sites

---

## Étape 2 : Démarrage des services Docker

### Commande utilisée :

```bash
cd /home/quentindv/frappe_docker

docker compose \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  up -d
```

### Services démarrés :
- `db` : MariaDB 11.8
- `redis-cache` : Redis pour cache
- `redis-queue` : Redis pour queue
- `configurator` : Configuration initiale (one-time)
- `backend` : Serveur Gunicorn (Python/Frappe)
- `frontend` : Serveur Nginx (port 8080)
- `websocket` : Socket.io pour temps réel
- `queue-short` : Worker pour tâches courtes
- `queue-long` : Worker pour tâches longues
- `scheduler` : Planificateur de tâches

### Vérification des services :

```bash
docker compose ps
```

---

## Étape 3 : Création du site erp.hexalith.com

### Commande :

```bash
docker compose exec -T backend bench new-site erp.hexalith.com \
  --mariadb-root-password "Protege est le mot de passe de la db" \
  --admin-password "Il pleut constamment dehors c'est ennuyeux" \
  --install-app erpnext
```

**Identifiants de connexion :**
- URL : https://erp.hexalith.com
- Utilisateur : Administrator
- Mot de passe : Il pleut constamment dehors c'est ennuyeux

### Résultat :
- Site créé : `/home/frappe/frappe-bench/sites/erp.hexalith.com/`
- Base de données : `_3e95d93e86ce5cef` (nom généré automatiquement)
- Utilisateur admin : Administrator
- Apps installées : frappe, erpnext

### Avertissement :
```
Warning: MariaDB version ['11.8', '5'] is more than 10.8 which is not yet tested with Frappe Framework.
```
(Cet avertissement peut être ignoré, MariaDB 11.8 fonctionne correctement)

---

## Étape 4 : Création du site vente.hexalith.com

### Commande :

```bash
docker compose exec -T backend bench new-site vente.hexalith.com \
  --mariadb-root-password "Protege est le mot de passe de la db" \
  --admin-password "L Hiver est presque arrive a son terme" \
  --install-app erpnext
```

**Identifiants de connexion :**
- URL : https://vente.hexalith.com
- Utilisateur : Administrator
- Mot de passe : L Hiver est presque arrive a son terme

### Résultat :
- Site créé : `/home/frappe/frappe-bench/sites/vente.hexalith.com/`
- Base de données : Nouvelle base dédiée
- Utilisateur admin : Administrator
- Apps installées : frappe, erpnext

---

## Étape 5 : Installation de l'app erpnext-real-estate-sales (PROBLÈMES)

### URL du dépôt (privé) :
🔒 git@github.com:Itaneo/erpnext-real-estate-sales.git (Dépôt privé - nécessite clé SSH)

### État actuel :
✅ Dépôt Git cloné avec succès dans `apps/vente_immo/`
✅ SSH configuré dans le conteneur
✅ App ajoutée à `sites/apps.txt`
✅ Installation pip effectuée
❌ Problème de structure de l'app empêchant l'installation sur le site

### Problème rencontré :

L'app `vente_immo` a une structure non standard qui empêche Frappe de la reconnaître correctement :

**Structure actuelle :**
```
apps/vente_immo/
├── __init__.py           # Contient __version__
├── hooks.py              # Configuration de l'app
├── modules.txt           # Liste "Real Estate"
├── real_estate/          # Module métier principal
│   ├── __init__.py
│   ├── doctype/
│   └── ...
└── vente_immo/           # Sous-répertoire mal configuré
    ├── __init__.py
    └── workspace/
```

**Erreur :**
```
ModuleNotFoundError: No module named 'vente_immo'
```

### Solutions possibles :

#### Solution 1 : Correction manuelle de la structure (Recommandé)

```bash
# Entrer dans le conteneur
docker compose exec backend bash

# Déplacer tous les fichiers de real_estate/ vers vente_immo/
cd apps/vente_immo
cp -r real_estate/* vente_immo/
# Vérifier que vente_immo/__init__.py existe et contient les imports nécessaires

# Réinstaller avec pip
python -m pip install -e .

# Redémarrer les services
exit
docker compose restart backend queue-short queue-long scheduler

# Installer sur le site
docker compose exec -T backend bench --site vente.hexalith.com install-app vente_immo

# Reconstruire les assets
docker compose exec -T backend bench build

# Redémarrer frontend
docker compose restart frontend websocket
```

#### Solution 2 : Utiliser une image Docker custom

Créer une image Docker personnalisée avec l'app pré-installée (plus complexe mais plus propre pour la production).

#### Solution 3 : Contacter le développeur de l'app

L'app semble avoir des problèmes de packaging. Il serait préférable de contacter le développeur pour corriger la structure.

### Commandes déjà effectuées :

```bash
# 1. Installation de SSH dans le conteneur
docker compose exec -u root -T backend apt-get update && apt-get install -y openssh-client

# 2. Configuration SSH
docker cp ~/.ssh/id_ed25519 frappe_docker-backend-1:/home/frappe/.ssh/
docker compose exec -T backend chmod 600 ~/.ssh/id_ed25519
docker compose exec -T backend ssh-keyscan github.com >> ~/.ssh/known_hosts

# 3. Clonage du dépôt
docker compose exec -T backend bench get-app erpnext-real-estate-sales git@github.com:Itaneo/erpnext-real-estate-sales.git

# 4. Correction du setup.py (problème d'import de __version__)
# Fichier modifié pour définir __version__ = "0.0.1" directement

# 5. Création de vente_immo/__init__.py
echo '__version__ = "0.0.1"' > apps/vente_immo/vente_immo/__init__.py

# 6. Installation pip
python -m pip install -e apps/vente_immo/

# 7. Ajout à apps.txt
echo 'vente_immo' >> sites/apps.txt

# 8. Redémarrage des services
docker compose restart backend queue-short queue-long scheduler
```

---

## Étape 6 : Configuration force_https (TERMINÉ)

### Commandes exécutées :

```bash
# Retirer vente_immo de apps.txt (problème de structure)
docker compose exec -T backend bash -c "grep -v vente_immo sites/apps.txt > sites/apps.txt.tmp && mv sites/apps.txt.tmp sites/apps.txt"

# Redémarrer les services backend
docker compose restart backend queue-short queue-long scheduler

# Site erp.hexalith.com
docker compose exec -T backend bench --site erp.hexalith.com set-config force_https 1

# Site vente.hexalith.com
docker compose exec -T backend bench --site vente.hexalith.com set-config force_https 1
```

### Ou modifier manuellement les fichiers :

**Fichier** : `sites/erp.hexalith.com/site_config.json`
```json
{
  "force_https": 1,
  ...
}
```

**Fichier** : `sites/vente.hexalith.com/site_config.json`
```json
{
  "force_https": 1,
  ...
}
```

---

## Étape 7 : Configuration Nginx (Déjà configurée)

### Script utilisé :
`~/install-nginx-hexalith.sh`

### Domaines configurés :
- `erp.hexalith.com` → 127.0.0.1:8080
- `vente.hexalith.com` → 127.0.0.1:8080

### Fonctionnalités :
- HTTPS/SSL activé
- Redirection HTTP → HTTPS
- Optimisations SSL (TLS 1.2/1.3, OCSP Stapling, etc.)
- Headers de sécurité

---

## Commandes utiles

### Gestion des sites

```bash
# Lister tous les sites
docker compose exec backend bench --site all list-apps

# Voir les apps installées sur un site spécifique
docker compose exec backend bench --site erp.hexalith.com list-apps

# Accéder au shell du site
docker compose exec backend bench --site erp.hexalith.com console

# Backup d'un site
docker compose exec backend bench --site erp.hexalith.com backup

# Backup de tous les sites
docker compose exec backend bench --site all backup
```

### Gestion des services

```bash
# Voir les logs
docker compose logs -f backend
docker compose logs -f frontend

# Redémarrer un service
docker compose restart backend
docker compose restart frontend

# Arrêter tous les services
docker compose down

# Arrêter et supprimer les volumes (ATTENTION: perte de données)
docker compose down -v
```

### Gestion des apps

```bash
# Installer une app sur un site
docker compose exec backend bench --site <SITE_NAME> install-app <APP_NAME>

# Désinstaller une app d'un site
docker compose exec backend bench --site <SITE_NAME> uninstall-app <APP_NAME>

# Reconstruire les assets
docker compose exec backend bench build

# Reconstruire pour une app spécifique
docker compose exec backend bench build --app <APP_NAME>

# Migrer un site après une mise à jour
docker compose exec backend bench --site <SITE_NAME> migrate
```

### Maintenance

```bash
# Entrer dans le conteneur backend
docker compose exec backend bash

# Vérifier l'état de la base de données
docker compose exec backend bench --site erp.hexalith.com mariadb

# Clear cache d'un site
docker compose exec backend bench --site erp.hexalith.com clear-cache
```

---

## Vérification du fonctionnement

### 1. Vérifier que les services sont actifs :

```bash
docker compose ps
```

Tous les services doivent être "Up".

### 2. Vérifier que les sites sont accessibles localement :

```bash
curl -H "Host: erp.hexalith.com" http://localhost:8080
curl -H "Host: vente.hexalith.com" http://localhost:8080
```

### 3. Tester via navigateur (après configuration DNS) :

- https://erp.hexalith.com → Doit afficher ERPNext
- https://vente.hexalith.com → Doit afficher ERPNext (avec Real Estate une fois l'app installée)

### 4. Vérifier la redirection HTTPS :

```bash
curl -I http://erp.hexalith.com
# Doit rediriger vers https://erp.hexalith.com
```

---

## Configuration DNS requise

Pour que les domaines fonctionnent en production :

```
erp.hexalith.com     A    <IP_DU_SERVEUR>
vente.hexalith.com   A    <IP_DU_SERVEUR>
```

Ou avec wildcard :

```
*.hexalith.com       A    <IP_DU_SERVEUR>
```

---

## Problèmes rencontrés et solutions

### 1. Erreur d'authentification MariaDB

**Problème** : `Access denied for user 'root'@'172.21.0.7'`

**Cause** : Le volume db-data contenait une ancienne base avec un mot de passe différent.

**Solution** :
```bash
docker compose down -v  # Supprimer tous les volumes
docker compose up -d     # Redémarrer avec la nouvelle config
```

### 2. Dépôt GitHub introuvable

**Problème** : https://github.com/Itaneo/erpnext-real-estate-sales retourne 404

**Action nécessaire** : Obtenir l'URL correcte du dépôt avant de continuer l'installation.

---

## Résumé final - État de la configuration

### ✅ Étapes complétées

1. ✅ **Fichier .env créé** avec configuration multitenancy
2. ✅ **Services Docker démarrés** (MariaDB, Redis, ERPNext)
3. ✅ **Site erp.hexalith.com créé** avec ERPNext de base
4. ✅ **Site vente.hexalith.com créé** avec ERPNext de base
5. ✅ **force_https configuré** sur les deux sites
6. ✅ **Services redémarrés** et vérifiés

### ⏳ Étapes en attente

7. ⏳ **Installer erpnext-real-estate-sales** sur vente.hexalith.com (problème de structure de l'app à résoudre)
8. ⏳ **Tests finaux** via navigateur web
9. ⏳ **Configuration DNS** (si pas déjà fait)

### 🌐 Sites accessibles

- **https://erp.hexalith.com**
  - ERPNext standard
  - Admin: Administrator
  - Password: Il pleut constamment dehors c'est ennuyeux
  - Status: ✅ Opérationnel

- **https://vente.hexalith.com**
  - ERPNext standard (app Real Estate à installer)
  - Admin: Administrator
  - Password: L Hiver est presque arrive a son terme
  - Status: ✅ Opérationnel

### 📊 Services Docker actifs

```bash
docker compose ps
```

Tous les services doivent être "Up" :
- ✅ db (MariaDB 11.8)
- ✅ redis-cache
- ✅ redis-queue
- ✅ backend (Gunicorn)
- ✅ frontend (Nginx - port 8080)
- ✅ websocket (Socket.io)
- ✅ queue-short
- ✅ queue-long
- ✅ scheduler

### 🔧 Prochaines étapes pour installer l'app Real Estate

L'app `erpnext-real-estate-sales` a été clonée mais rencontre des problèmes de structure. Pour l'installer :

**Option recommandée :** Contacter le développeur de l'app pour corriger la structure du package.

**Alternative temporaire :** Suivre la "Solution 1" dans l'[Étape 5](#étape-5--installation-de-lapp-erpnext-real-estate-sales-problèmes) pour réorganiser manuellement la structure de l'app.

---

## Contacts et ressources

- Documentation Frappe Docker : https://github.com/frappe/frappe_docker
- Documentation ERPNext : https://docs.erpnext.com
- Forum Frappe : https://discuss.frappe.io

---

*Document créé le : 2025-11-19*
*Dernière mise à jour : 2025-11-19*
