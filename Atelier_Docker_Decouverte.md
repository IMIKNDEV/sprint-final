# Atelier Docker — Premier conteneur : un Laravel neuf qui tourne dans Docker

**Durée totale : ~2h30** (installation comprise)

> ⚠️ **Règle : on ne touche PAS au projet fil rouge aujourd'hui.**
> Cet atelier se fait sur un **projet Laravel neuf et jetable**. On n'installe pas un nouvel environnement au milieu d'un projet noté. L'atelier sert à apprendre Docker à côté, sans risque.

> 💻 **Machine faible ou moins de 10 Go libres ?** N'installe pas. Mets-toi **en binôme** avec quelqu'un qui installe. Tu manipules avec lui. Le but aujourd'hui c'est de comprendre, pas de remplir ton disque.

---

## Objectif

À la fin de l'atelier :
- Docker installé
- Une **image** de ton application construite avec un **Dockerfile**
- Un **conteneur** qui tourne
- La page d'accueil Laravel visible dans ton navigateur, servie depuis le conteneur

Et tu dois savoir expliquer chaque terme et chaque étape. Pas juste copier-coller.

## Les termes (à lire avant de taper quoi que ce soit)

- **Image** : le paquet figé qui contient tout : l'OS de base, PHP, ton code, les dépendances. Une image ne s'exécute pas. Elle se stocke et se partage.
- **Conteneur** : une image en train de tourner. Il est isolé et jetable. Tu peux le détruire et en relancer un identique en 2 secondes.
- **Dockerfile** : le fichier texte qui décrit comment construire l'image, étape par étape. Il se versionne dans git.
- **Port mapping** (`-p 8000:8000`) : le conteneur est isolé. Cette option dit : ce qui arrive sur le port 8000 de ma machine part vers le port 8000 du conteneur. Sans ça, l'app tourne mais tu ne peux pas la voir.
- **Docker Hub** : le "GitHub des images". C'est de là que viennent les images de base (`php:8.3-cli`, `composer:2`). Compte pas obligatoire aujourd'hui : on télécharge des images publiques.

---

## Étape 0 — Installer Docker Desktop (30–45 min, téléchargement compris)

1. Télécharge **Docker Desktop** : https://www.docker.com/products/docker-desktop/
2. Sur **Windows** : l'installeur active **WSL2** (une mini-machine Linux dans Windows — Docker en a besoin car les conteneurs sont des processus Linux). Accepte, redémarre si demandé.
3. Lance Docker Desktop. Attends que la baleine en bas à gauche soit **verte / "Engine running"**.
4. Vérifie dans un terminal :
```bash
docker --version
docker run hello-world
```
`hello-world` télécharge une micro-image de test et l'exécute. Si tu vois "Hello from Docker!", tout marche. C'est ton premier conteneur.

**Poids de l'installation :** Docker Desktop + WSL2 = 3 à 5 Go. C'est pour ça que les machines pleines passent en binôme.

---

## Étape 1 — Créer le projet Laravel jetable (10 min)

```bash
composer create-project laravel/laravel docker-decouverte
cd docker-decouverte
php artisan serve
```
Ouvre http://localhost:8000 : la page d'accueil Laravel s'affiche. **Arrête le serveur (Ctrl+C).**

Là, l'app tourne avec TON PHP, sur TA machine. La suite de l'atelier : la faire tourner avec le PHP du conteneur, comme si ta machine n'avait pas PHP du tout.

---

## Étape 2 — Écrire le Dockerfile (30 min — l'étape où on comprend)

À la racine du projet, crée un fichier nommé exactement `Dockerfile` (sans extension) :

```dockerfile
# 1. L'image de base : un Linux avec PHP 8.3 déjà installé
FROM php:8.3-cli

# 2. L'image de base est minimaliste : on installe git et unzip,
#    dont Composer a besoin pour télécharger les paquets
RUN apt-get update && apt-get install -y git unzip && rm -rf /var/lib/apt/lists/*

# 3. On récupère Composer depuis son image officielle
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# 4. Le dossier de travail À L'INTÉRIEUR du conteneur
WORKDIR /app

# 5. On copie le code du projet dans le conteneur
COPY . .

# 6. On installe les dépendances DANS le conteneur
RUN composer install --no-interaction

# 7. Préparer Laravel : le .env et la clé d'application
RUN cp .env.example .env && php artisan key:generate

# 8. Sessions et cache en mode fichier (pas de base de données aujourd'hui)
ENV SESSION_DRIVER=file CACHE_STORE=file

# 9. Le port que le conteneur expose
EXPOSE 8000

# 10. La commande lancée au démarrage du conteneur
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

**Lis chaque ligne et son commentaire.** Deux points importants :
- `FROM php:8.3-cli` : tout le monde part du même PHP, peu importe ce qui est installé sur sa machine. C'est ça qui règle le "ça marche chez moi".
- `--host=0.0.0.0` : par défaut `artisan serve` n'écoute que l'intérieur du conteneur. `0.0.0.0` lui dit d'accepter les connexions qui viennent de ta machine.

Crée aussi un fichier `.dockerignore` (même logique que `.gitignore` : ce qu'on ne copie pas dans l'image) :
```
vendor
node_modules
.git
.env
```
On ignore `vendor` parce que le `composer install` se fait DANS le conteneur. Copier ton vendor local pourrait ramener des dépendances incompatibles.

---

## Étape 3 — Construire l'image (10 min)

```bash
docker build -t mon-laravel .
```
- `build` : construit l'image en suivant le Dockerfile, ligne par ligne.
- `-t mon-laravel` : le nom (tag) de ton image.
- `.` : "le Dockerfile est ici".

La première fois c'est long (téléchargement de l'image PHP ~500 Mo, puis composer install). Regarde le terminal : tu vois ton Dockerfile s'exécuter ligne par ligne.

**Vérifie :**
```bash
docker images
```
`mon-laravel` apparaît dans la liste, avec sa taille. Dans Docker Desktop, onglet **"Images"** : elle est là aussi. Une image, pas encore de conteneur.

---

## Étape 4 — Lancer le conteneur (10 min)

```bash
docker run -p 8000:8000 --name mon-app mon-laravel
```
- `run` : crée un conteneur à partir de l'image et le démarre.
- `-p 8000:8000` : le pont entre ta machine et le conteneur.
- `--name mon-app` : un nom pour le retrouver.

**Ouvre http://localhost:8000 : la page d'accueil Laravel s'affiche.**

Ce qui vient de se passer : cette page est servie par un PHP qui n'est pas celui de ta machine. Elle tournerait pareil sur le PC de ton voisin ou sur un serveur Azure, sans rien installer d'autre que Docker.

---

## Étape 5 — Le tour de Docker Desktop (15 min)

Ouvre Docker Desktop, onglet **"Containers"** :
1. Tu vois `mon-app`, statut **Running**, avec le port `8000:8000` cliquable (il ouvre le navigateur).
2. Clique sur le conteneur, onglet **Logs** : tu vois les requêtes arriver en direct. Recharge la page du navigateur et regarde les lignes apparaître.
3. Bouton **Stop** : le conteneur s'arrête. Recharge le navigateur : la page ne répond plus. Le conteneur est éteint, l'image existe toujours.
4. Bouton **Start** : il repart, la page revient.
5. Bouton **Delete** (après Stop) : le conteneur est détruit. L'image, elle, est intacte (onglet Images). Relance un conteneur neuf :
```bash
docker run -p 8000:8000 --name mon-app-2 mon-laravel
```
La page revient, dans un conteneur neuf. C'est ça, "l'image est figée, le conteneur est jetable".

---

## Critères de réussite

- `docker run hello-world` fonctionne
- Ton `Dockerfile` existe et tu sais expliquer chaque ligne à l'oral
- `docker images` montre `mon-laravel`
- http://localhost:8000 affiche la page Laravel servie par le conteneur
- Tu as fait le cycle Stop / Start / Delete / re-run dans Docker Desktop
- Tu sais répondre : "quelle est la différence entre l'image et le conteneur ?" et "à quoi sert `-p 8000:8000` ?"

## En cas de blocage

- **Docker Desktop ne démarre pas / erreur WSL2 (Windows)** : lance `wsl --update` dans PowerShell en admin, redémarre. Si la virtualisation est désactivée dans le BIOS, passe en binôme pour aujourd'hui.
- **`docker build` échoue sur composer install** : vérifie que tu es à la racine du projet (là où sont `composer.json` et le `Dockerfile`).
- **`port is already allocated`** : un autre process occupe le 8000 (ton `php artisan serve` de l'étape 1 ?). Arrête-le, ou change le pont : `-p 8001:8000` puis http://localhost:8001.
- **Page blanche / erreur 500** : regarde les **Logs** dans Docker Desktop. C'est le premier réflexe, toujours. Si l'erreur parle de sessions ou de base de données, vérifie la ligne `ENV SESSION_DRIVER=file CACHE_STORE=file` du Dockerfile.
- **`Cannot connect to the Docker daemon`** : Docker Desktop n'est pas lancé (la baleine doit être verte).
- **Un changement de code n'apparaît pas** : normal. L'image a été construite AVANT ton changement, elle est figée. Il faut rebuild (`docker build ...`) puis relancer un conteneur. C'est le concept le plus important de la journée.

## Et docker-compose ?

Aujourd'hui, un seul conteneur suffit (pas de base de données : la page welcome n'en a pas besoin). docker-compose sert quand il y a **plusieurs conteneurs** à lancer ensemble (l'app + MySQL + Redis...). C'est le prochain atelier. Et c'est exactement ce que fait Laravel Sail sous le capot pour ceux qui l'utilisent.

### Bonus (si tu as fini en avance)
Crée un `docker-compose.yml` minimal qui lance ton app :
```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
```
Puis :
```bash
docker compose up
```
Même résultat qu'avant, mais décrit dans un fichier au lieu d'une commande à rallonge. Ajoute un service `mysql` si tu veux explorer (image `mysql:8.0`, et cherche pourquoi il lui faut un **volume**).

---

## Timeframe récapitulatif

| Étape | Durée |
|---|---|
| 0. Installation Docker Desktop + hello-world | 30–45 min |
| 1. Projet Laravel jetable | 10 min |
| 2. Dockerfile (lecture + écriture) | 30 min |
| 3. Build de l'image | 10 min |
| 4. Run + page dans le navigateur | 10 min |
| 5. Tour de Docker Desktop (stop/start/delete/logs) | 15 min |
| Marge (blocages, binômes) | 20–30 min |
| **Total** | **~2h15–2h30** |
