# Deploiement ModelBench sur le VPS

Runbook d'installation initiale. A executer une seule fois, dans l'ordre, directement sur le VPS.

## 1. Utilisateur de deploiement

```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
```

## 2. Cle SSH dediee au pipeline CI/CD

Sur votre poste, generer une paire de cles dediee :

```bash
ssh-keygen -t ed25519 -f modelbench_deploy_key -C "deploiement-modelbench" -N ""
```

Deposer la cle publique sur le VPS :

```bash
sudo mkdir -p /home/deploy/.ssh
sudo tee -a /home/deploy/.ssh/authorized_keys < modelbench_deploy_key.pub
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

Dans les reglages GitHub de **modelbench** et de **modelbench_frontend** (Settings > Secrets and
variables > Actions), creer dans **chacun** des deux repos :

| Secret | Valeur |
|---|---|
| `VPS_HOST` | adresse IP ou nom d'hote du VPS |
| `VPS_USER` | `deploy` |
| `VPS_SSH_KEY` | contenu du fichier prive `modelbench_deploy_key` |

## 3. Copier la configuration de deploiement sur le VPS

Depuis votre poste, a la racine du workspace :

```bash
scp -r deploy/ deploy@<VPS_HOST>:/tmp/deploy-modelbench
ssh deploy@<VPS_HOST> "sudo mv /tmp/deploy-modelbench /opt/modelbench && sudo chown -R deploy:deploy /opt/modelbench"
```

## 4. Fichier `.env` reel, uniquement sur le VPS

```bash
ssh deploy@<VPS_HOST>
cd /opt/modelbench
cp .env.example .env
```

Editer `.env` et remplacer les deux valeurs `change-me` par :
- un mot de passe Postgres robuste pour `DB_PASSWORD`,
- une chaine aleatoire d'au moins 64 caracteres pour `JWT_SECRET` (par exemple
  `openssl rand -base64 48`).

Ce fichier ne quitte jamais le VPS et n'est jamais ajoute a un depot git.

## 5. Modules et vhosts Apache

```bash
sudo a2enmod proxy proxy_http
sudo cp /opt/modelbench/apache/api.conf /etc/apache2/sites-available/mbapi.golden-technologies.com.conf
sudo cp /opt/modelbench/apache/app.conf /etc/apache2/sites-available/modelbench.golden-technologies.com.conf
sudo a2ensite mbapi.golden-technologies.com.conf
sudo a2ensite modelbench.golden-technologies.com.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## 6. DNS

Pointer les enregistrements A (ou AAAA) de `modelbench.golden-technologies.com` et
`mbapi.golden-technologies.com` vers l'adresse IP du VPS, et attendre la propagation avant l'etape
suivante.

## 7. HTTPS avec certbot

```bash
sudo apt update && sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d modelbench.golden-technologies.com -d mbapi.golden-technologies.com
```

Certbot reecrit les deux vhosts pour servir en HTTPS et ajoute la redirection HTTP vers HTTPS.

## 8. Premier lancement

```bash
cd /opt/modelbench
docker compose pull
docker compose up -d
docker compose ps
```

Verifier que la colonne `PORTS` du service `postgres` ne montre aucun mapping vers l'hote.

## 9. Rendre les images GHCR publiques

Une fois que chaque workflow CI a pousse sa premiere image (apres le premier push sur `main` des
deux repos) : dans GitHub, Profil > Packages > `modelbench_backend`, puis `modelbench_frontend` >
Package settings > Change visibility > Public. Necessaire une seule fois par image : evite d'avoir a
configurer `docker login ghcr.io` sur le VPS.

## Verification finale

- `https://modelbench.golden-technologies.com` affiche la page de connexion.
- `https://mbapi.golden-technologies.com/swagger` affiche Swagger UI.
- Connexion avec `admin` / `admin123`, creation d'un dataset, verifie que le CRUD fonctionne de bout
  en bout a travers Apache et les deux conteneurs.
