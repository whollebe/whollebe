https://career-fair.42lausanne.ch/dashboard


# Documentation Django — ORM et backend

> **Rôle de Django dans le projet** : Django fait **tout le backend** : il est à la fois
> - l'**ORM** de la base PostgreSQL (définition des tables, migrations, requêtes) ;
> - le **serveur HTTP** de l'API (`/api/...`, voir `routes.md`) ;
> - le serveur **WebSocket** du temps réel (matchs, amis en ligne, notifications), via **Django Channels** ;
> - le **client de Judge0** (compilation et tests du code des joueurs) et du LLM/RAG.
>


## 1. Commandes d'installation

```bash
python -m venv env                      # crée un environnement virtuel nommé "env"
source env/bin/activate                 # l'active (Linux/Mac)
pip install django "psycopg[binary]"    # Django + driver PostgreSQL pour Python (psycopg 3)
pip install python-dotenv               # lecture du fichier .env
pip install django-ninja                # couche API (routes JSON, validation, doc OpenAPI)
pip install channels daphne channels-redis   # WebSockets (ASGI) + couche de messages via Redis
pip install httpx                       # appels HTTP asynchrones (Judge0, LLM)
pip install pillow                      # nécessaire pour ImageField (avatar)
pip freeze > requirements.txt           # liste des dépendances (utile pour Docker)

django-admin startproject config .      # crée le PROJET "config" dans le dossier courant (le "." évite un dossier en plus)
python manage.py startapp core          # APPLICATION "core" : modèles + logique métier (services)
python manage.py startapp api           # APPLICATION "api"  : les routes HTTP (Django Ninja)
python manage.py startapp realtime      # APPLICATION "realtime" : WebSockets (Channels)
```

Puis : ajouter `'core'`, `'api'` et `'realtime'` dans `INSTALLED_APPS` de `config/settings.py` (voir section 3).

### Projet vs application (à ne pas confondre)

| | Projet (`config`) | Applications (`core`, `api`, `realtime`) |
|---|---|---|
| C'est quoi ? | La configuration globale | Des modules fonctionnels |
| Contient | `settings.py`, `urls.py`, `asgi.py`... | `core` : `models.py`, `services.py`, `migrations/` · `api` : les routes · `realtime` : les consumers WebSocket |
| Nombre | 1 seul | Autant qu'on veut (ici 3) |

> Seule `core` a des modèles (donc des migrations). `api` et `realtime` n'ont **pas de tables** : elles *utilisent* `core`.

### Qui crée quoi ? (base, schéma, tables)

| Étape | Qui s'en occupe | Comment |
|---|---|---|
| Lancer le serveur PostgreSQL | Nous (Docker) | image `postgres` dans le `docker-compose.yml` |
| Créer la **base** (vide) `transcendence` | PostgreSQL, au tout premier démarrage | grâce aux variables `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` (ou `CREATE DATABASE` à la main) |
| **Décrire** les tables | Nous | dans `core/models.py` |
| Générer le plan des changements | Django | `python manage.py makemigrations` (crée des fichiers dans `core/migrations/`) |
| **Créer** les tables dans la base | Django | `python manage.py migrate` |

À retenir :
- `models.py` décrit le schéma ; `makemigrations` puis `migrate` **créent les tables à l'intérieur d'une base qui existe déjà**. Django ne crée pas la base elle-même.
- La configuration de PostgreSQL (ses fichiers, ou les variables de l'image Docker) ne contient **rien** sur le schéma de nos tables.
- Le schéma vit donc dans **`models.py` + les fichiers de migration**, tous deux versionnés dans git. C'est aussi ce qu'on décrira dans la section « Database Schema » du README.

### Commandes de tous les jours

```bash
python manage.py makemigrations   # détecte les changements dans models.py et génère un fichier de migration
python manage.py migrate          # applique les migrations sur la base PostgreSQL
python manage.py runserver        # lance le serveur de dev (avec 'daphne' dans INSTALLED_APPS : HTTP + WebSocket)
python manage.py shell            # console Python avec Django chargé (pratique pour tester l'ORM)
python manage.py dbshell          # ouvre psql directement sur la base
python manage.py createsuperuser  # crée un admin (pour /admin)
python manage.py check            # vérifie que la configuration est valide
python manage.py test             # lance les tests (voir core/tests.py)
```

---

## 2. Arborescence

```
.
├── .env                     ← secrets réels (JAMAIS dans git)
├── .env.example             ← mêmes variables, sans les vraies valeurs (DANS git)
├── .gitignore
├── docker-compose.yml
├── manage.py
├── config/                  ← le PROJET Django (configuration)
│   ├── __init__.py
│   ├── asgi.py              ← point d'entrée ASGI : HTTP + WebSocket (section 4.3)
│   ├── settings.py
│   ├── urls.py              ← branche l'API Ninja sur /api/ et /admin
│   └── wsgi.py              ← non utilisé (on sert en ASGI)
├── core/                    ← APPLICATION : données et logique métier
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations/
│   │   └── __init__.py
│   ├── management/commands/ ← à créer : seed_exercises.py, seed_topics.py...
│   ├── models.py            ← les tables (section 5)
│   ├── services.py          ← à créer : logique métier (ELO, fin de match, amis, succès...)
│   ├── tests.py
│   └── views.py             ← non utilisé
├── api/                     ← APPLICATION : les routes HTTP de routes.md (section 4)
│   ├── api.py               ← crée l'objet NinjaAPI et y branche les routeurs
│   ├── auth.py              ← /api/auth/*
│   ├── users.py             ← /api/users/*
│   ├── friends.py           ← /api/friends/*
│   ├── leaderboard.py       ← /api/leaderboard/*
│   ├── catalog.py           ← /api/topics, /api/exercises/:id
│   ├── learn.py             ← /api/learn/*, /api/exercises/:id/run|submit
│   ├── matches.py           ← /api/matches/*
│   ├── schemas.py           ← formes des données entrantes et sortantes (validation)
│   └── judge0.py            ← client asynchrone de Judge0
└── realtime/                ← APPLICATION : temps réel
    ├── consumers.py         ← la connexion WebSocket d'un joueur
    ├── routing.py           ← URL des WebSockets (/ws/)
    └── events.py            ← fonction notify() : envoyer un événement à un joueur
```

> L'organisation finale du dépôt (par exemple tout le backend dans un dossier `backend/`) est à décider en équipe. **Règle d'architecture** : `api/` et `realtime/` sont « fines » (elles reçoivent, vérifient, répondent) ; **la logique métier vit dans `core/services.py`**.

---

## 3. Fichier par fichier

Pour chaque fichier : **À quoi il sert**, **Ce qu'on peut faire dedans** (avec exemples), **Utile pour notre projet**, et un indicateur d'utilité : 🟢 on l'utilise beaucoup · 🟡 on l'utilise un peu / optionnel · 🔴 inutile pour nous.

---

### `manage.py` (racine) 🟢

**À quoi il sert**
Script de ligne de commande de Django, **point d'entrée de toutes les commandes** `python manage.py <commande>`. À l'intérieur, il définit la variable d'environnement `DJANGO_SETTINGS_MODULE = "config.settings"` (pour que Django sache où est la config), puis appelle `execute_from_command_line(sys.argv)` qui lance la commande demandée.

**Ce qu'on peut faire avec**

```bash
python manage.py makemigrations            # génère les fichiers de migration à partir de models.py
python manage.py makemigrations --dry-run  # montre ce qui serait généré, sans écrire
python manage.py migrate                   # applique les migrations sur PostgreSQL
python manage.py migrate core 0001   # revient (ou avance) à une migration précise
python manage.py showmigrations            # liste les migrations [X] appliquées / [ ] non appliquées
python manage.py sqlmigrate core 0001  # affiche le SQL exact d'une migration
python manage.py shell                     # console Python avec Django chargé (tester l'ORM)
python manage.py dbshell                   # ouvre psql directement sur la base
python manage.py inspectdb                 # génère des modèles à partir de tables SQL existantes
python manage.py flush                     # VIDE toutes les tables (garde le schéma) ⚠️
python manage.py dumpdata core > data.json   # exporte les données en JSON
python manage.py loaddata data.json        # importe des données JSON (fixtures)
python manage.py check                     # vérifie la configuration
```

**Utile pour notre projet**
- Il y a surtout `makemigrations` + `migrate` à chaque changement de modèle.
- `dumpdata` / `loaddata` permettent de partager un jeu d'exercices de test entre membres de l'équipe.
- On peut y ajouter **nos propres commandes** (ex. `python manage.py seed_exercises` pour remplir la table `Exercise`) en créant `core/management/commands/seed_exercises.py`.
- **On ne modifie jamais `manage.py`.**

---

### `config/settings.py` 🟢

**À quoi il sert**
**Fichier de configuration global** du projet. Django le lit une seule fois au démarrage ; toutes les variables en MAJUSCULES sont accessibles partout via `from django.conf import settings`.

**Ce qu'on peut faire dedans**

> 📌 **Comment lire cette partie** : le point 1 (`.env` + `DATABASES`) est **obligatoire**. Les points 2, 3 et 4 sont indépendants ; le point 4 (`AUTH_USER_MODEL`) est **à trancher en équipe avant la première migration**.

*1. Configurer PostgreSQL avec un fichier `.env` (le plus important)*

Le sujet exige que les identifiants soient stockés dans un `.env` ignoré par git, avec un `.env.example` fourni. On le fait donc **dès le départ**, ce qui montrera aussi un historique git propre.

**Étape 1 : créer `.env.example`** (versionné dans git, sans vraies valeurs)

```bash
# Django
DJANGO_SECRET_KEY=change-me
DEBUG=1

# PostgreSQL (les 3 premières variables sont aussi lues par l'image Docker postgres)
POSTGRES_DB=transcendence
POSTGRES_USER=transcendence
POSTGRES_PASSWORD=change-me
POSTGRES_HOST=localhost     # dans Docker : le nom du service PostgreSQL (ex: db)
POSTGRES_PORT=5432

# Backend
ALLOWED_HOSTS=localhost,127.0.0.1
CSRF_TRUSTED_ORIGINS=https://localhost
REDIS_URL=redis://localhost:6379/0     # dans Docker : redis://redis:6379/0

# Services externes
JUDGE0_URL=http://judge0:2358
```

**Étape 2 : chacun copie ce fichier en `.env`** et met ses vraies valeurs (`cp .env.example .env`).

**Étape 3 : ignorer `.env` dans git** : ajouter `.env` dans `.gitignore`, **avant** le tout premier commit qui le contiendrait. Si un secret est déjà parti dans git, il faut le considérer comme compromis et le changer.

**Étape 4 : lire les variables dans `config/settings.py`**, en remplaçant le bloc `DATABASES` SQLite généré par Django :

```python
import os
from pathlib import Path
from dotenv import load_dotenv

BASE_DIR = Path(__file__).resolve().parent.parent
load_dotenv(BASE_DIR / ".env")      # charge le .env en local ; dans Docker, les variables sont déjà injectées

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME':     os.environ.get("POSTGRES_DB", "transcendence"),
        'USER':     os.environ.get("POSTGRES_USER", "transcendence"),
        'PASSWORD': os.environ["POSTGRES_PASSWORD"],      # pas de valeur par défaut : erreur claire si absent
        'HOST':     os.environ.get("POSTGRES_HOST", "localhost"),
        'PORT':     os.environ.get("POSTGRES_PORT", "5432"),
    }
}
```

- `os.environ.get("NOM", "défaut")` lit la variable `NOM`, ou prend la valeur par défaut si elle n'existe pas.
- `os.environ["NOM"]` (avec crochets) lève une `KeyError` si la variable manque. On le garde pour les **secrets** : mieux vaut un plantage clair qu'un mot de passe par défaut oublié.
- `load_dotenv()` ne remplace pas une variable déjà définie dans l'environnement : Docker garde donc la priorité sur le fichier.

*2. Même principe pour `SECRET_KEY` et `DEBUG`*

```python
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DEBUG", "0") == "1"      # "1" → True, autre chose → False (donc faux par défaut)
```

*3. (Facultatif) Ajouter ses propres constantes, accessibles depuis tout le code*

```python
ELO_START = 1000
ELO_K_FACTOR = 32
JUDGE0_URL = os.environ.get("JUDGE0_URL", "http://judge0:2358")
```

```python
from django.conf import settings
settings.ELO_START        # 1000
```

*4. Le modèle utilisateur : `AUTH_USER_MODEL` (décidé : option B, à faire avant la 1re migration)*

**Le contexte.** Django possède déjà sa propre table d'utilisateurs (`auth_user`) avec : `username`, `password` (déjà haché), `email`, `first_name`, `last_name`, `is_active`, `is_staff`, `is_superuser`, `last_login`, `date_joined`, plus des tables de groupes et de permissions. La question était : *utilise-t-on ça, l'étend-on, ou crée-t-on le nôtre ?*

| Option | Principe | Pour | Contre |
|---|---|---|---|
| **A. User « maison » simple** | `class User(models.Model)`, sans `AUTH_USER_MODEL` | Contrôle total | Il faudrait gérer hachage, sessions et tokens à la main ; deux tables « utilisateur » ; plus de connexion à `/admin` |
| **B. `AbstractUser`** ✅ | `class User(AbstractUser)` + nos champs (`elo`, `github`...) + `AUTH_USER_MODEL` | **Une seule** table ; hachage **salé** fourni ; **sessions, login/logout, `/admin`, WebSocket authentifié** fonctionnent tout seuls ; très simple | Hérite de champs inutiles (`first_name`, `is_staff`...) ; difficile à changer après coup |
| **C. `AbstractBaseUser`** | Modèle minimal, on écrit le reste | Minimal | Plus de code (un *manager* obligatoire), plus de pièges |
| **D. `Profile` lié à `auth.User`** | Nos champs dans une table `Profile` en `OneToOneField` | Aucun risque pour la 1re migration | Deux tables à joindre à chaque lecture d'un joueur |

**Décision : B.** Avant, FastAPI devait gérer l'authentification ; maintenant que Django est le backend, **tout** le système d'authentification de Django est utilisable (le sujet exige des mots de passe « hashés et salés » : `create_user` / `make_password` le font sans rien brancher). Le schéma complet du modèle est dans la section 5.

```python
# config/settings.py
AUTH_USER_MODEL = 'core.User'
```

```python
# core/models.py (extrait ; version complète en section 5)
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    email = models.EmailField(unique=True)      # chez AbstractUser, l'email n'est PAS unique par défaut
    github = models.CharField(max_length=100, blank=True)
    elo = models.IntegerField(default=1000)
    # username, password, is_active, last_login, date_joined... sont déjà fournis
```

```python
# Créer un utilisateur : le mot de passe est haché et salé automatiquement
user = User.objects.create_user(username="alice", email="a@a.com", password="motdepasse")

# ⚠️ User.objects.create(..., password="motdepasse") stockerait le mot de passe EN CLAIR : toujours create_user.

# Vérifier un mot de passe : on passe par authenticate() (voir section 4.5)
from django.contrib.auth import authenticate
authenticate(username="alice", password="motdepasse")     # renvoie l'utilisateur, ou None
```

⚠️ **À faire avant la toute première `migrate`.** Si `AUTH_USER_MODEL` est ajouté après, les migrations de `auth` et `admin` sont déjà liées à l'ancien modèle : il faut alors supprimer la base et les migrations, et recommencer. De plus, `'core'` doit être dans `INSTALLED_APPS`.

*5. Les réglages propres au backend (HTTP, WebSocket, sessions)*

```python
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")
CSRF_TRUSTED_ORIGINS = os.environ.get("CSRF_TRUSTED_ORIGINS", "https://localhost").split(",")

# Cookies de session : jamais lisibles par le JavaScript, et HTTPS seulement en production
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = not DEBUG
CSRF_COOKIE_SECURE = not DEBUG

# ASGI : c'est config/asgi.py qui sert HTTP ET WebSocket (voir section 4.3)
ASGI_APPLICATION = "config.asgi.application"

# Channels : la "poste" qui permet d'envoyer un message à un joueur depuis n'importe où dans le code
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {"hosts": [os.environ.get("REDIS_URL", "redis://localhost:6379/0")]},
    }
}

# Avatars envoyés par les joueurs (PUT /api/users/me/avatar)
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

- Le **frontend et l'API doivent être servis sous le même domaine** (via le reverse proxy : `/` → frontend, `/api/` et `/ws/` → backend). Ainsi le cookie de session et le CSRF fonctionnent sans configuration CORS.
- `channels_redis` demande un serveur Redis (un conteneur de plus dans `docker-compose.yml`, voir 4.10). Pour un simple test en local, `"BACKEND": "channels.layers.InMemoryChannelLayer"` suffit (ne fonctionne qu'avec **un seul** processus).

**Description de chaque variable générée**

| Variable | À quoi ça sert | Pour nous |
|---|---|---|
| `BASE_DIR` | Chemin absolu de la racine du projet (via `pathlib.Path`). Sert à construire d'autres chemins. | 🟡 |
| `SECRET_KEY` | Clé secrète pour la cryptographie (signatures, sessions). **Ne jamais la commit.** | 🟢 |
| `DEBUG` | `True` = pages d'erreur détaillées. **`False` en production obligatoirement.** | 🟢 |
| `ALLOWED_HOSTS` | Domaines autorisés à servir l'app (obligatoire si `DEBUG=False`). | 🟢 |
| `INSTALLED_APPS` | Applications actives (détail ci-dessous). | 🟢 |
| `MIDDLEWARE` | Composants qui traitent chaque requête HTTP (sessions, CSRF, authentification...). **Nécessaires** : ce sont eux qui identifient l'utilisateur connecté. | 🟢 |
| `ROOT_URLCONF` | Fichier d'URLs principal (`'config.urls'`). | 🟢 |
| `TEMPLATES` | Moteur de templates HTML (requis par l'admin ; on ne s'en sert pas pour notre front). | 🟡 |
| `WSGI_APPLICATION` | Objet WSGI (`config.wsgi.application`). **Non utilisé** : on sert en ASGI. | 🔴 |
| `ASGI_APPLICATION` | Objet ASGI (`config.asgi.application`) : HTTP + WebSocket. | 🟢 |
| `CHANNEL_LAYERS` | Où Channels fait transiter les messages entre joueurs (Redis). | 🟢 |
| `DATABASES` | **Connexion à la base.** | 🟢 |
| `AUTH_USER_MODEL` | Notre modèle `User` (point 4). | 🟢 |
| `AUTH_PASSWORD_VALIDATORS` | Règles de validation de mot de passe (longueur, mots courants...). À appliquer à l'inscription. | 🟢 |
| `LANGUAGE_CODE` | Langue par défaut (`'en-us'`, ou `'fr-fr'`). | 🟡 |
| `TIME_ZONE` | Fuseau horaire (`'UTC'` par défaut, ou `'Europe/Paris'`). | 🟡 |
| `USE_I18N` | Active la traduction. | 🟡 (le sujet demande plusieurs langues si le module i18n est visé) |
| `USE_TZ` | Si `True`, les dates sont stockées en UTC en base. **À laisser `True`** (indispensable pour `ends_at` des matchs). | 🟢 |
| `STATIC_URL` | Préfixe d'URL des fichiers statiques (utile pour l'admin). | 🟡 |
| `MEDIA_URL` / `MEDIA_ROOT` | Où sont stockés les fichiers envoyés (avatars). | 🟢 |
| `DEFAULT_AUTO_FIELD` | Type de la clé primaire automatique (`BigAutoField` = entier 64 bits). | 🟢 |

**`INSTALLED_APPS` — détail de chaque entrée**

```python
INSTALLED_APPS = [
    'daphne',                        # en PREMIER : fait de "runserver" un serveur ASGI (HTTP + WebSocket)
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'channels',
    'core',
    'api',
    'realtime',
]
```

| Entrée | À quoi ça sert | Tables créées en base | Pour nous |
|---|---|---|---|
| `daphne` | Serveur ASGI (HTTP + WebSocket). | aucune | 🟢 |
| `django.contrib.admin` | Interface web d'administration (`/admin`) pour voir/éditer la base. | `django_admin_log` | 🟡 Pratique pour créer les exercices et déboguer |
| `django.contrib.auth` | Utilisateurs, mots de passe hachés, groupes, permissions. | `auth_group`, `auth_permission`, et notre table `User` | 🟢 |
| `django.contrib.contenttypes` | Registre de tous les modèles existants. Requis par `auth` et `admin`. | `django_content_type` | 🟡 Dépendance interne |
| `django.contrib.sessions` | Sessions utilisateur (cookie de connexion) stockées en base. | `django_session` | 🟢 C'est notre authentification |
| `django.contrib.messages` | Messages « flash » (requis par l'admin). | aucune | 🟡 |
| `django.contrib.staticfiles` | Gestion CSS/JS/images (admin). | aucune | 🟡 |
| `channels` | WebSockets. | aucune | 🟢 |
| `'core'` | **Nos modèles et notre logique métier.** Sans elle, Django ignore notre `models.py`. | nos tables | 🟢 **Indispensable** |
| `'api'` | Les routes HTTP. | aucune | 🟢 |
| `'realtime'` | Les consumers WebSocket. | aucune | 🟢 |

> ⚠️ Il faut les noms des **applications** (`'core'`, `'api'`, `'realtime'`) et non `'config'` (nom du projet).
>
> 💡 Les apps `contrib` sont maintenant toutes utiles (l'admin a besoin de `messages` et `staticfiles`, l'authentification a besoin de `sessions`) : on les laisse.

**Utile pour notre projet**
- Mettre **tous les mots de passe et clés dans des variables d'environnement** (jamais en dur dans le code).
- Les constantes de jeu (ELO de départ, facteur K, durée d'un match, URL de Judge0) vivent ici pour que toute l'équipe les change au même endroit :

```python
MATCH_DURATION_SECONDS = 15 * 60       # durée d'un match 1vs1
INVITATION_TTL_SECONDS = 5 * 60        # une invitation expire au bout de 5 min
SUBMIT_PENALTY = 10                    # points perdus par "submit" (voir routes.md : "costs points")
```

---

### `config/urls.py` 🟢, `config/asgi.py` 🟢

**`config/urls.py`** : l'annuaire des URLs HTTP. On y branche l'API et l'admin :

```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path
from api.api import api                       # l'objet NinjaAPI (voir section 4.4)

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", api.urls),                   # toutes les routes de routes.md commencent par /api/
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)   # avatars, en dev seulement
```

**`config/asgi.py`** : point d'entrée du serveur (HTTP **et** WebSocket). On le complète en section 4.3.

### `core/models.py` 🟢🟢🟢 (le fichier le plus important pour nous)

**À quoi il sert**
**Décrit la structure de la base de données** : quelles tables existent, quelles colonnes elles ont, et comment elles sont reliées entre elles. On l'écrit en Python (des classes), et Django s'occupe de créer les vraies tables dans PostgreSQL.

> Ce fichier est un **plan**, pas la base elle-même. Écrire une classe dans `models.py` ne crée **rien** tant qu'on n'a pas lancé `makemigrations` puis `migrate` (voir la section `migrations/` plus bas).

---

#### Partie 0 — Les notions de base (à lire en premier)

**Une base de données, c'est quoi ?**
Un endroit où l'on range des informations de façon organisée et durable (un programme qui s'arrête ne les perd pas). Ici, c'est **PostgreSQL** : un programme qui tourne à côté du backend et qui stocke tout (joueurs, exercices, matchs...). Le backend Django lui « pose des questions » (*requêtes*) pour lire ou modifier les données.

**Une table, c'est quoi ?**
Imagine **une feuille Excel** :
- la **table** = la feuille (ex : la feuille des joueurs) ;
- une **colonne** = une propriété, avec un type fixe (ex : `username` = du texte, `elo` = un nombre) ;
- une **ligne** = un élément (ex : un joueur précis).

Exemple, la table des joueurs (`users`) :

| id | username | email | elo |
|---|---|---|---|
| 1 | alice | alice@mail.com | 1000 |
| 2 | bob | bob@mail.com | 1230 |
| 3 | carol | carol@mail.com | 1105 |

**La clé primaire (`id`)**
Chaque ligne a un **numéro unique** : son `id`. C'est ce qui permet de désigner « le joueur n°2 » sans ambiguïté (même si deux joueurs s'appelaient pareil). Django ajoute cette colonne **automatiquement**, on ne l'écrit pas.

**La clé étrangère (*foreign key*) : relier deux tables**
Au lieu de recopier les infos d'un joueur partout, on **stocke seulement son `id`** dans l'autre table. Exemple, la table des amitiés (`friends`) :

| id | stalker_id | followed_id |
|---|---|---|
| 1 | 1 | 2 |
| 2 | 3 | 1 |

Lecture : la ligne 1 dit « le joueur n°**1** (alice) suit le joueur n°**2** (bob) ». Si bob change de pseudo, on le change **à un seul endroit** (la table `users`) et toutes les amitiés restent correctes.

**Les 3 types de relations**

| Relation | Signification | Exemple chez nous | Outil Django |
|---|---|---|---|
| **1 → N** (un à plusieurs) | Une ligne A est liée à plusieurs lignes B | 1 exercice a plusieurs tests | `ForeignKey` (placée du côté « plusieurs ») |
| **N ↔ N** (plusieurs à plusieurs) | Chacun peut être lié à plusieurs de l'autre côté | 1 exercice a plusieurs thèmes, 1 thème a plusieurs exercices | `ManyToManyField` |
| **1 ↔ 1** (un à un) | Une ligne A ↔ exactement une ligne B | 1 joueur ↔ 1 profil étendu | `OneToOneField` |

**SQL vs ORM : le « traducteur »**
Pour parler à PostgreSQL, il existe un langage : le **SQL**. Un **ORM** (*Object-Relational Mapping*) est un traducteur : **on écrit du Python, Django génère le SQL à notre place** et nous renvoie des objets Python.

| On veut... | SQL (ce que PostgreSQL comprend) | ORM Django (ce qu'on écrit) |
|---|---|---|
| Trouver alice | `SELECT * FROM users WHERE username = 'alice';` | `User.objects.get(username="alice")` |
| Les 10 meilleurs | `SELECT * FROM users ORDER BY elo DESC LIMIT 10;` | `User.objects.order_by("-elo")[:10]` |
| Ajouter un joueur | `INSERT INTO users (username, email) VALUES ('dave', 'd@d.com');` | `User.objects.create(username="dave", email="d@d.com")` |
| Changer un ELO | `UPDATE users SET elo = 1050 WHERE id = 1;` | `user.elo = 1050; user.save()` |
| Supprimer | `DELETE FROM users WHERE id = 1;` | `user.delete()` |

**Correspondance à retenir**

| Côté Python (ORM) | Côté PostgreSQL |
|---|---|
| Une **classe** (`class User`) | Une **table** |
| Un **attribut** de la classe (`username = ...`) | Une **colonne** |
| Un **objet** (`User(username="alice")`) | Une **ligne** |

**Schéma vs données**
- Le **schéma** = la structure (quelles tables, quelles colonnes). On le décrit dans `models.py` et on le change avec les **migrations**.
- Les **données** = le contenu (alice, bob...). On les manipule avec les requêtes ORM.

---

#### Partie 1 — Premier modèle, pas à pas


> 📌 **Exemple pédagogique.** Ce premier `User` est volontairement minimal (`models.Model`) pour comprendre l'ORM. **Le vrai `User` du projet est un `AbstractUser`** (section 3, `settings.py`, point 4 ; schéma complet en section 5). Avec `AbstractUser`, créer un joueur se fait avec `User.objects.create_user(...)` (qui hache le mot de passe) et non `create(...)`.

Dans `core/models.py` :

```python
from django.db import models


class User(models.Model):
    username    = models.CharField(max_length=50, unique=True)
    email       = models.EmailField(unique=True)
    elo         = models.IntegerField(default=1000)
    in_game     = models.BooleanField(default=False)
    created_at  = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "users"     # nom de la table dans PostgreSQL (voir encadré plus bas)
```

**Ce que chaque ligne signifie**

| Ligne | Effet |
|---|---|
| `class User(models.Model)` | « Je décris une table. » Hériter de `models.Model` est obligatoire : c'est ce qui donne à la classe tous les pouvoirs de l'ORM (`.objects`, `.save()`, `.delete()`...). |
| `username = models.CharField(max_length=50, unique=True)` | Colonne `username` : texte de 50 caractères max, deux joueurs ne peuvent pas avoir le même. |
| `email = models.EmailField(unique=True)` | Colonne texte, vérifie que ça ressemble à un email, unique. |
| `elo = models.IntegerField(default=1000)` | Colonne nombre entier. Si on ne précise rien à la création, elle vaut 1000. |
| `in_game = models.BooleanField(default=False)` | Colonne vrai/faux. |
| `created_at = models.DateTimeField(auto_now_add=True)` | Colonne date+heure, **remplie automatiquement** à la création de la ligne. |
| `class Meta: db_table = "users"` | Nomme la table `users`. |

**Ce que Django fait tout seul**
- Il ajoute la colonne **`id`** (clé primaire, auto-incrémentée : 1, 2, 3...).
- Il choisit le nom de la table (par défaut `core_user` ; modifiable avec `db_table`).

**Le SQL équivalent** (ce que Django enverra à PostgreSQL, simplifié) :

```sql
CREATE TABLE users (
    id          bigint PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    username    varchar(50) NOT NULL UNIQUE,
    email       varchar(254) NOT NULL UNIQUE,
    elo         integer NOT NULL,
    in_game     boolean NOT NULL,
    created_at  timestamp with time zone NOT NULL
);
```

**Utiliser le modèle** (après `makemigrations` + `migrate`), dans `python manage.py shell` :

```python
>>> from core.models import User

>>> alice = User.objects.create(username="alice", email="alice@mail.com")
>>> alice.id
1
>>> alice.elo               # valeur par défaut
1000
>>> alice.created_at        # rempli automatiquement
datetime.datetime(2026, 10, 19, 14, 3, 12, tzinfo=datetime.timezone.utc)

>>> User.objects.count()
1

>>> alice.elo = 1030
>>> alice.save()            # écrit la modification en base (UPDATE)

>>> User.objects.get(username="alice").elo     # on relit depuis la base
1030

>>> User.objects.create(username="alice", email="autre@mail.com")
IntegrityError: duplicate key value violates unique constraint ...   # unique=True protège la base

>>> alice.delete()          # supprime la ligne
```

---

#### Partie 2 — Les types de champs : lequel choisir ?

| Champ | Type PostgreSQL | Quand l'utiliser | Exemple chez nous |
|---|---|---|---|
| `CharField(max_length=N)` | `VARCHAR(N)` | Texte **court** dont on connaît la taille max (**`max_length` obligatoire**) | `username`, `nom` d'exercice |
| `TextField()` | `TEXT` | Texte **long**, sans limite | `description`, `test_code`, `reference_solution` |
| `IntegerField()` | `INTEGER` | Nombre entier (jusqu'à ~2 milliards) | `elo`, `score1`, `attempts1` |
| `BigIntegerField()` | `BIGINT` | Très grands entiers | rarement utile |
| `FloatField()` | `DOUBLE PRECISION` | Nombre à virgule | un taux de réussite |
| `BooleanField()` | `BOOLEAN` | Vrai / faux | `in_game`, `debug`, `pointers` |
| `DateTimeField()` | `TIMESTAMPTZ` | Date + heure | `created_at`, `ends_at` d'un match |
| `DateField()` | `DATE` | Date seule | rarement utile |
| `EmailField()` | `VARCHAR(254)` | Email (avec vérification) | `email` |
| `URLField()` | `VARCHAR(200)` | Lien (avec vérification) | `avatar` |
| `JSONField()` | `JSONB` | Données structurées (liste, dictionnaire) | `hints`, `authorized` (liste de fonctions autorisées) |
| `ForeignKey(...)` | `BIGINT` + contrainte | Lien « plusieurs vers un » | `Test.exercise` |
| `ManyToManyField(...)` | table intermédiaire cachée | Lien « plusieurs vers plusieurs » | `Exercise.topics` |
| `OneToOneField(...)` | `BIGINT UNIQUE` + contrainte | Lien « un vers un » | rarement utile |

**Conseil** : en cas de doute entre `CharField` et `TextField`, on prend `TextField` pour du code ou des descriptions, `CharField` pour des noms et identifiants.

---

#### Partie 3 — Les options des champs

Les options se mettent entre les parenthèses du champ.

| Option | Signification | Exemple |
|---|---|---|
| `default=...` | Valeur par défaut si on n'en donne pas | `elo = IntegerField(default=1000)` |
| `unique=True` | Interdit deux lignes avec la même valeur | `username` |
| `null=True` | La colonne peut contenir **NULL** (= « aucune valeur ») **en base** | `time1 = IntegerField(null=True)` : pas encore de temps tant que le match n'est pas fini |
| `blank=True` | Le champ peut être **laissé vide** dans les validations (formulaires, admin) | `github = CharField(blank=True)` |
| `db_index=True` | Crée un **index** pour accélérer les recherches/tris sur cette colonne | `elo` (leaderboard) |
| `choices=...` | Limite aux valeurs d'une liste | `status` d'un match |
| `auto_now_add=True` | Date remplie **à la création** de la ligne, jamais modifiée ensuite | `created_at` |
| `auto_now=True` | Date mise à jour **à chaque `save()`** | `updated_at` |
| `max_length=N` | Taille max (obligatoire pour `CharField`) | `max_length=50` |

**`null` vs `blank` : la confusion classique**
- `null=True` → concerne la **base de données** : la colonne accepte la valeur NULL.
- `blank=True` → concerne la **validation** : le champ n'est pas obligatoire.
- Règle simple : pour un **texte** optionnel, utiliser `blank=True` (et laisser une chaîne vide `""`, pas de `null`). Pour un **nombre, une date ou une relation** optionnels, utiliser `null=True` (et souvent `blank=True` aussi).

**C'est quoi un index ?**
Comme l'index à la fin d'un livre : sans lui, PostgreSQL doit lire **toutes** les lignes pour trouver celles qui correspondent à la recherche ; avec lui, il va directement au bon endroit. À mettre sur les colonnes qu'on **filtre ou trie souvent** (ELO pour le classement). `unique=True` crée déjà un index. Trop d'index ralentit les écritures : pas la peine d'en mettre partout.

---

#### Partie 4 — Les relations entre tables

##### 4.1 `ForeignKey` : 1 → N (un exercice a plusieurs tests)

```python
class Exercise(models.Model):
    nom = models.CharField(max_length=100)
    description = models.TextField()

    class Meta:
        db_table = "exercises"


class Test(models.Model):
    exercise = models.ForeignKey(Exercise, on_delete=models.CASCADE, related_name="tests")
    input = models.TextField(blank=True)
    expected_out = models.TextField()

    class Meta:
        db_table = "tests"
```

- La `ForeignKey` se met dans la table du côté **« plusieurs »** (un test appartient à **un** exercice, donc `Test` porte la clé).
- En base, Django crée la colonne **`exercise_id`** (nom du champ + `_id`) qui contient l'`id` de l'exercice.
- Les deux sens fonctionnent en Python :

```python
test = Test.objects.get(id=5)
test.exercise.nom               # du test vers son exercice → "ft_strlen"

ex = Exercise.objects.get(nom="ft_strlen")
ex.tests.all()                  # de l'exercice vers tous ses tests (grâce à related_name="tests")
ex.tests.count()
```

- **`related_name`** : le nom de l'accès « en sens inverse ». Sans lui, Django invente `ex.test_set.all()`. Avec `related_name="tests"`, on écrit `ex.tests.all()` : plus lisible.

##### 4.2 `on_delete` : que se passe-t-il quand on supprime la ligne référencée ?

`on_delete` est **obligatoire** sur une `ForeignKey`. Il répond à : « si je supprime cet exercice, que deviennent ses tests ? »

| Valeur | Effet | Exemple chez nous |
|---|---|---|
| `CASCADE` | Les lignes liées sont **supprimées aussi** | Un joueur supprimé → ses amitiés supprimées. Un exercice supprimé → ses tests supprimés. |
| `PROTECT` | **Interdit** la suppression (erreur) tant qu'il reste des lignes liées | Un exercice déjà utilisé dans des matchs ne doit pas disparaître (l'historique casserait). |
| `SET_NULL` | La clé de la ligne liée passe à NULL (**nécessite `null=True`**) | Un match garde sa trace même si un joueur supprime son compte. |

##### 4.3 Deux `ForeignKey` vers la même table (le cas `Game`)

Un match a **deux joueurs**, donc deux clés vers `User`. Dans ce cas, **`related_name` devient obligatoire** et doit être **différent** pour chacune, sinon Django ne sait pas quel « sens inverse » est lequel :

```python
class Game(models.Model):
    exercise   = models.ForeignKey(Exercise, on_delete=models.PROTECT, related_name="games")
    challenger = models.ForeignKey(User, on_delete=models.CASCADE, related_name="games_as_challenger")
    victim     = models.ForeignKey(User, on_delete=models.CASCADE, related_name="games_as_victim")
    score1     = models.IntegerField(default=0)
    score2     = models.IntegerField(default=0)

    class Meta:
        db_table = "games"
```

```python
alice.games_as_challenger.all()      # les matchs lancés par alice
alice.games_as_victim.all()          # les matchs où alice a été défiée
```

##### 4.4 `ManyToManyField` : N ↔ N (exercices ↔ thèmes)

Un exercice peut avoir plusieurs thèmes (pointeurs **et** chaînes) et un thème regroupe plusieurs exercices. On ne peut pas mettre ça dans une seule colonne : Django crée **une table intermédiaire cachée** qui stocke les paires (exercice, thème).

```python
class Topic(models.Model):
    name = models.CharField(max_length=50, unique=True)

    class Meta:
        db_table = "topics"


class Exercise(models.Model):
    nom    = models.CharField(max_length=100)
    topics = models.ManyToManyField(Topic, related_name="exercises", blank=True)
```

```python
pointers = Topic.objects.get(name="pointers")
ex = Exercise.objects.get(nom="ft_strlen")

ex.topics.add(pointers)                   # lie l'exercice au thème
ex.topics.all()                           # tous les thèmes de l'exercice
pointers.exercises.all()                  # tous les exercices du thème
Exercise.objects.filter(topics__name="pointers")    # filtrer par thème (route GET /api/topics...)
ex.topics.remove(pointers)                # retire le lien
```

##### 4.5 Une relation N ↔ N « à la main » (le cas `Friend`)

On pourrait faire `ManyToManyField` sur `User` pour les amis, mais on a besoin de **retenir qui a demandé à qui** (« stalker » → « followed ») et, plus tard, l'état de la demande (en attente / acceptée). On crée donc **notre propre table de liaison** :

```python
class Friend(models.Model):
    stalker  = models.ForeignKey(User, on_delete=models.CASCADE, related_name="following")
    followed = models.ForeignKey(User, on_delete=models.CASCADE, related_name="followers")

    class Meta:
        db_table = "friends"
        constraints = [
            models.UniqueConstraint(fields=["stalker", "followed"], name="uniq_friend_pair"),   # impossible de s'ajouter 2 fois
        ]
```

```python
Friend.objects.create(stalker=alice, followed=bob)    # alice suit bob

alice.following.all()      # les lignes Friend où alice est "stalker"
bob.followers.all()        # les lignes Friend où bob est "followed"

# Obtenir directement les UTILISATEURS que suit alice :
User.objects.filter(followers__stalker=alice)
```

---

#### Partie 5 — `choices` : limiter les valeurs possibles

```python
class Game(models.Model):
    class Status(models.TextChoices):
        PENDING  = "pending",  "En attente"
        READY    = "ready",    "Prêt"
        PLAYING  = "playing",  "En cours"
        FINISHED = "finished", "Terminé"

    status = models.CharField(max_length=10, choices=Status.choices, default=Status.PENDING)
```

- La partie gauche (`"pending"`) est **ce qui est stocké en base** ; la partie droite (`"En attente"`) est le texte lisible (admin).
- On s'en sert dans le code sans écrire de texte à la main (donc sans faute de frappe) :

```python
game.status = Game.Status.PLAYING
game.save()

Game.objects.filter(status=Game.Status.FINISHED)
```

⚠️ Django **ne vérifie pas** `choices` au moment du `save()` (seulement dans les formulaires/admin). C'est donc notre code qui doit utiliser les bonnes valeurs.

---

#### Partie 6 — La classe `Meta` : réglages de la table

```python
class Meta:
    db_table = "friends"                          # nom de la table en base
    constraints = [models.UniqueConstraint(fields=["stalker", "followed"], name="uniq_friend_pair")]   # combinaison unique
    ordering = ["-id"]                            # tri par défaut (le "-" = décroissant)
    indexes = [models.Index(fields=["followed"])] # index supplémentaire
```

| Option | Rôle |
|---|---|
| `db_table` | Nom de la table (optionnel : par défaut `<app>_<modèle>`, ex : `core_user`) |
| `constraints` + `UniqueConstraint` | Empêche les doublons sur une **combinaison** de colonnes (une paire d'amis). Remplace l'ancien `unique_together`. |
| `ordering` | Ordre par défaut des résultats (sans `order_by` explicite) |
| `indexes` | Index supplémentaires pour accélérer les recherches |
| `constraints` (autres) | Règles vérifiées par la base : `CheckConstraint` (ex : « `elo` ≥ 0 ») |

---

#### Partie 7 — Ajouter des méthodes aux modèles

Un modèle est une classe Python : on peut y mettre des fonctions qui concernent **une ligne**.

```python
class User(models.Model):
    ...

    def __str__(self):                          # texte affiché quand on "print" l'objet (shell, admin)
        return f"{self.username} ({self.elo})"

    def win_probability_against(self, other):
        """Formule ELO : probabilité que self batte other (entre 0 et 1)."""
        return 1 / (1 + 10 ** ((other.elo - self.elo) / 400))
```

```python
>>> print(alice)
alice (1000)
>>> alice.win_probability_against(bob)      # alice 1000 vs bob 1230
0.2268...
```

---

#### Partie 8 — Faire des requêtes (le quotidien du backend)

Tout passe par **`Modèle.objects`** (le « gestionnaire » qui discute avec la base).

##### 8.1 Lire des données

| Méthode | Renvoie | Détail |
|---|---|---|
| `.get(...)` | **1 seul objet** | Erreur `User.DoesNotExist` s'il n'y en a aucun, `MultipleObjectsReturned` s'il y en a plusieurs |
| `.filter(...)` | Une liste de résultats (peut être vide) | Ne plante jamais |
| `.exclude(...)` | Tous sauf ceux qui correspondent | Inverse de `filter` |
| `.all()` | Toutes les lignes | |
| `.order_by("-elo")` | Résultats triés | `"-"` = décroissant |
| `.first()` / `.last()` | 1 objet ou `None` | Pas d'erreur si vide |
| `.count()` | Un nombre | |
| `.exists()` | `True` / `False` | Plus rapide que `count() > 0` |
| `[:10]` | Les 10 premiers | Devient `LIMIT 10` en SQL |
| `.values("username", "elo")` | Liste de **dictionnaires** | Pratique pour renvoyer du JSON |

```python
# Gérer l'absence de résultat (route GET /api/users/:username)
try:
    user = User.objects.get(username="inconnu")
except User.DoesNotExist:
    ...   # renvoyer une erreur 404 (HttpError(404, ...) avec Django Ninja)

# Ou, plus court, sans exception :
user = User.objects.filter(username="inconnu").first()     # None si absent
```

##### 8.2 Les filtres : `champ__opération`

Avec **deux underscores**, on précise comment comparer :

| Écriture | Signification | SQL |
|---|---|---|
| `username="alice"` | égal (par défaut) | `username = 'alice'` |
| `username__iexact="ALICE"` | égal sans tenir compte des majuscules | |
| `username__icontains="eli"` | contient (sans casse) | `ILIKE '%eli%'` |
| `username__startswith="al"` | commence par | |
| `elo__gt=1200` / `elo__gte=1200` | supérieur / supérieur ou égal | `>` / `>=` |
| `elo__lt=1200` / `elo__lte=1200` | inférieur / inférieur ou égal | `<` / `<=` |
| `id__in=[1, 2, 3]` | dans la liste | `IN (1, 2, 3)` |
| `time1__isnull=True` | est NULL | `IS NULL` |

```python
User.objects.filter(elo__gte=1100, in_game=False)       # plusieurs conditions = ET
User.objects.filter(username__icontains="eli")          # GET /api/users?search=eli
User.objects.order_by("-elo")[:50]                      # GET /api/leaderboard/users (top 50)
```

**Traverser une relation** (avec `__` aussi) :

```python
Game.objects.filter(exercise__nom="ft_strlen")          # matchs dont l'exercice s'appelle ft_strlen
Game.objects.filter(challenger__username="alice")       # matchs lancés par alice
Exercise.objects.filter(topics__name="pointers")        # exercices du thème "pointers"
```

##### 8.3 Combiner avec OU : l'objet `Q`

`filter(a, b)` fait un **ET**. Pour un **OU**, on utilise `Q` :

```python
from django.db.models import Q

# Historique d'alice : matchs où elle est challenger OU victim
Game.objects.filter(Q(challenger=alice) | Q(victim=alice))
```

##### 8.4 Écrire des données

```python
# Créer
user = User.objects.create(username="dave", email="d@d.com")

# Modifier (méthode 1 : un objet)
user.elo = 1050
user.save()                              # écrit TOUTES les colonnes
user.save(update_fields=["elo"])         # écrit seulement la colonne elo (plus sûr)

# Modifier (méthode 2 : plusieurs lignes d'un coup)
User.objects.filter(in_game=True).update(in_game=False)

# Supprimer
user.delete()

# Créer seulement si ça n'existe pas déjà
topic, created = Topic.objects.get_or_create(name="algo")     # created = True/False
```

##### 8.5 `F` : modifier une valeur en se basant sur la valeur actuelle

```python
from django.db.models import F

User.objects.filter(id=1).update(elo=F("elo") + 15)
```

**Pourquoi pas `user.elo += 15; user.save()` ?** Imagine que deux matchs d'alice se terminent à la même seconde :
- avec `user.elo += 15` : les deux lisent `elo = 1000`, les deux écrivent `1015` → **un gain est perdu** ;
- avec `F("elo") + 15` : c'est **PostgreSQL** qui fait l'addition (`SET elo = elo + 15`) → les deux sont comptés, `1030`.

##### 8.6 Compter, moyenner : `aggregate` et `annotate`

```python
from django.db.models import Avg, Count

Exercise.objects.aggregate(Avg("elo"))                    # un seul résultat : {"elo__avg": 1043.5}
Exercise.objects.annotate(nb_games=Count("games"))        # ajoute à CHAQUE exercice un champ nb_games
Exercise.objects.annotate(nb_games=Count("games")).order_by("-nb_games")   # les plus joués d'abord
```

- `aggregate` → **un résultat** pour toute la table.
- `annotate` → **un résultat par ligne**.

##### 8.7 Les requêtes sont « paresseuses » (*lazy*)

```python
qs = User.objects.filter(elo__gte=1100)    # RIEN n'est envoyé à la base ici
qs = qs.order_by("-elo")                   # toujours rien
for u in qs:                               # ICI la requête part
    print(u.username)
```

Django n'interroge la base que quand on **utilise** le résultat (boucle, `list()`, `len()`, affichage...). Ça permet de construire une requête en plusieurs étapes sans coût.

##### 8.8 Le problème « N+1 » et `select_related` / `prefetch_related`

```python
games = Game.objects.all()[:50]
for g in games:
    print(g.exercise.nom)          # ❌ 1 requête pour les matchs + 50 requêtes (une par exercice)
```

Chaque `g.exercise` retourne en base chercher l'exercice → **51 requêtes** au lieu de 1. Solution :

```python
games = Game.objects.select_related("exercise", "challenger", "victim")[:50]
for g in games:
    print(g.exercise.nom)          # ✅ 1 seule requête (avec jointure)
```

| Outil | Pour quel cas | Exemple |
|---|---|---|
| `select_related("champ")` | `ForeignKey` / `OneToOne` (on va **vers un seul** objet) | `Game` → `exercise` |
| `prefetch_related("champ")` | Sens inverse et `ManyToMany` (on va vers **plusieurs** objets) | `Exercise` → `tests`, `topics` |

---

#### Partie 9 — Les transactions : « tout ou rien »

Analogie : un virement bancaire retire 50 € d'un compte **et** les ajoute à l'autre. Si le programme plante au milieu, il ne faut surtout pas que l'argent ait disparu. Une **transaction** garantit : soit **toutes** les opérations réussissent, soit **aucune** n'est appliquée.

Chez nous : à la fin d'un match, on modifie l'ELO des **deux** joueurs et le match lui-même. Si ça plante après le premier joueur, la base serait incohérente.

```python
from django.db import transaction
from django.conf import settings


def finish_match(game_id, winner_id):
    with transaction.atomic():                                      # début de la transaction
        game = Game.objects.select_for_update().get(id=game_id)     # verrouille la ligne (personne d'autre ne la modifie en même temps)
        p1 = User.objects.select_for_update().get(id=game.challenger_id)
        p2 = User.objects.select_for_update().get(id=game.victim_id)

        score_p1 = 1 if winner_id == p1.id else 0                   # 1 = victoire, 0 = défaite
        delta = round(settings.ELO_K_FACTOR * (score_p1 - p1.win_probability_against(p2)))

        p1.elo += delta
        p2.elo -= delta
        p1.save(update_fields=["elo"])
        p2.save(update_fields=["elo"])

        game.status = Game.Status.FINISHED
        game.save(update_fields=["status"])
    # fin du "with" : tout est validé. Si une exception a été levée dedans : tout est annulé (rollback).
```

---

#### Partie 10 — Erreurs courantes

| Message d'erreur | Cause probable | Solution |
|---|---|---|
| `relation "users" does not exist` / `column ... does not exist` | On a modifié `models.py` mais **oublié** `makemigrations` + `migrate` | Lancer les deux commandes |
| `No changes detected` | `'core'` absent de `INSTALLED_APPS`, ou aucun changement réel | Vérifier `settings.py` |
| `User.DoesNotExist` | `.get()` n'a rien trouvé | Utiliser `try/except` ou `.filter(...).first()` |
| `MultipleObjectsReturned` | `.get()` a trouvé plusieurs lignes | Filtrer sur un champ `unique` (ex : `id`) |
| `IntegrityError: duplicate key ...` | Violation d'un `unique=True` ou `unique_together` | Vérifier avant de créer, ou `get_or_create` |
| `IntegrityError: null value in column ... violates not-null constraint` | Un champ obligatoire n'a pas reçu de valeur | Donner une valeur, un `default`, ou mettre `null=True` |
| `ProtectedError` | Suppression bloquée par `on_delete=PROTECT` | Supprimer d'abord les lignes liées, ou changer la règle |
| `SynchronousOnlyOperation` | ORM classique (`.get()`, `.save()`, boucle `for`, accès à `game.exercise`) dans une fonction `async def` | Méthodes `aget`, `acreate`, `async for`, ou `sync_to_async` (section 4.2) |
| `Reverse accessor ... clashes with ...` | Deux `ForeignKey` vers la même table sans `related_name` distinct | Ajouter des `related_name` différents |
| `FATAL: sorry, too many clients already` / `connection already closed` | Connexions à PostgreSQL non fermées, typiquement dans un consumer WebSocket | `database_sync_to_async` (et non `sync_to_async`) dans `realtime/` (section 4.9) |
| `django.core.exceptions.AppRegistryNotReady` | Import d'un modèle trop tôt (dans `asgi.py`, avant `get_asgi_application()`) | Respecter l'ordre de la section 4.3 |

**Utile pour notre projet**
- Après **chaque** modification de ce fichier : `makemigrations` puis `migrate`.
- Noms de projet et d'app en minuscules ; `db_table` soit partout, soit nulle part (cohérence).
- Ne jamais stocker un mot de passe en clair : seulement un **hash** (fait par `create_user` / `make_password` de Django).
- Pour les mises à jour de compteurs (ELO, scores), préférer `F("elo") + 15`, ou une transaction avec `select_for_update()` quand plusieurs champs sont liés.
- Noms de champs en `snake_case` (`total_score`), noms de modèles en `CamelCase` **au singulier** (`User`, pas `Users`).
- Mettre le code de **calcul** (ELO, fin de match) dans des fonctions Python normales (ex : `core/services.py`) plutôt que dans les routes de `api/` : plus facile à tester et à réutiliser.

---

### `core/migrations/` 🟢

**À quoi il sert**
C'est l'**historique de tous les changements de structure** de la base, comme « git », mais pour les tables. Chaque fichier (`0001_initial.py`, `0002_user_github.py`...) décrit **une étape** : « créer la table X », « ajouter la colonne Y »...

**Pourquoi c'est nécessaire**
Modifier `models.py` ne change **pas** la base PostgreSQL. Il faut un moyen de dire à la base : « voilà exactement ce qui a changé ». Et comme **toute l'équipe** a sa propre base locale, tout le monde doit pouvoir **rejouer les mêmes étapes dans le même ordre**. Les migrations servent à ça.

**L'analogie de la maison**

| Élément | Rôle |
|---|---|
| `models.py` | Le **plan** de la maison |
| `makemigrations` | L'architecte compare le nouveau plan à l'ancien et écrit un **ordre de travaux** (le fichier de migration) |
| `migrate` | Les **ouvriers** exécutent les ordres de travaux sur la vraie maison |
| La base PostgreSQL | La **maison** |

**Le cycle à retenir**

```
1. Modifier models.py
2. python manage.py makemigrations     → crée un fichier dans migrations/
3. (relire le fichier généré)
4. python manage.py migrate            → applique sur PostgreSQL
5. git add + commit (models.py ET le fichier de migration)
```

Le sous-fichier `migrations/__init__.py` est vide : il marque le dossier comme package Python.

---

#### Pas à pas : la première fois

**1. Prérequis** : PostgreSQL tourne, la base `transcendence` existe (c'est vous qui la créez), `DATABASES` est configuré, `'core'` est dans `INSTALLED_APPS`, et `models.py` contient au moins un modèle (voir Partie 1).

**2. `makemigrations`** : Django compare `models.py` aux migrations existantes (aucune pour l'instant) :

```bash
$ python manage.py makemigrations
Migrations for 'core':
  core/migrations/0001_initial.py
    + Create model User
```

Un fichier vient d'apparaître. **Rien n'a encore été modifié dans PostgreSQL.**

**3. Lire le fichier généré** (`core/migrations/0001_initial.py`) :

```python
from django.db import migrations, models


class Migration(migrations.Migration):

    initial = True                      # c'est la toute première migration de l'app

    dependencies = []                   # elle ne dépend d'aucune autre

    operations = [                      # la liste des travaux à faire
        migrations.CreateModel(
            name='User',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('username', models.CharField(max_length=50, unique=True)),
                ('email', models.EmailField(max_length=254, unique=True)),
                ('elo', models.IntegerField(default=1000)),
                ('in_game', models.BooleanField(default=False)),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
            options={
                'db_table': 'users',
            },
        ),
    ]
```

Comment le lire :

| Élément | Signification |
|---|---|
| `initial = True` | Première migration de l'application |
| `dependencies` | Migrations à appliquer **avant** celle-ci (l'ordre est garanti grâce à ça) |
| `operations` | Les actions : `CreateModel`, `AddField`, `RemoveField`, `AlterField`, `DeleteModel`, `RunPython`... |
| `CreateModel` | « Crée une table » avec la liste de ses colonnes |

**4. `migrate`** : applique réellement sur PostgreSQL :

```bash
$ python manage.py migrate
Operations to perform:
  Apply all migrations: core, admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
  Applying core.0001_initial... OK
  Applying sessions.0001_initial... OK
```

> 💡 La **première fois**, `migrate` crée aussi les tables des apps `django.contrib` listées dans `INSTALLED_APPS` (`auth_user`, `django_session`, `django_admin_log`...). C'est normal : ça explique pourquoi on voit plein de tables qu'on n'a pas écrites.

**5. Vérifier dans PostgreSQL** :

```bash
$ python manage.py dbshell
transcendence=# \dt                 -- liste les tables
transcendence=# \d users            -- détaille la table users (colonnes, types, contraintes)
transcendence=# SELECT * FROM users;
transcendence=# \q                  -- quitter
```

Django a aussi une table **`django_migrations`** : c'est son carnet de bord, il y note chaque migration déjà appliquée (c'est ce qui l'empêche de la rejouer). `SELECT * FROM django_migrations;` permet de la consulter.

---

#### Pas à pas : modifier un modèle plus tard

Exemple : on ajoute la colonne `github` à `User`.

```python
class User(models.Model):
    ...
    github = models.CharField(max_length=100, blank=True, default="")   # nouvelle ligne
```

```bash
$ python manage.py makemigrations
Migrations for 'core':
  core/migrations/0002_user_github.py
    + Add field github to user
```

Le nouveau fichier (simplifié) :

```python
class Migration(migrations.Migration):
    dependencies = [('core', '0001_initial')]       # s'applique APRÈS la 0001

    operations = [
        migrations.AddField(
            model_name='user',
            name='github',
            field=models.CharField(blank=True, default='', max_length=100),
        ),
    ]
```

Puis `python manage.py migrate` → PostgreSQL exécute `ALTER TABLE users ADD COLUMN github ...`.

**La question que Django peut poser**
Si on ajoute une colonne **obligatoire sans valeur par défaut** (ni `null=True`) alors que la table contient déjà des lignes, Django ne sait pas quoi mettre dedans :

```
It is impossible to add a non-nullable field 'github' to user without specifying a default.
 1) Provide a one-off default now
 2) Quit and manually define a default value in models.py
```

- Choix **1** : on tape une valeur à mettre dans les lignes existantes (ex : `''`).
- Choix **2** : on quitte et on ajoute `default=...` ou `null=True` dans `models.py`. **C'est la meilleure option** (la règle est alors écrite dans le code).

---

#### Les commandes utiles

| Commande | Rôle |
|---|---|
| `python manage.py makemigrations` | Crée les fichiers de migration d'après `models.py` |
| `python manage.py makemigrations --dry-run` | Montre ce qui serait créé, sans rien écrire |
| `python manage.py makemigrations --check` | Échoue s'il manque une migration (utile en CI) |
| `python manage.py makemigrations --name seed_topics` | Donne un nom à la migration |
| `python manage.py migrate` | Applique toutes les migrations non appliquées |
| `python manage.py showmigrations` | Liste les migrations : `[X]` appliquée, `[ ]` pas encore |
| `python manage.py sqlmigrate core 0001` | Affiche le **SQL exact** d'une migration, sans l'exécuter |
| `python manage.py migrate core 0001` | **Revient en arrière** jusqu'à la migration 0001 (annule les suivantes) |
| `python manage.py migrate core zero` | Annule **toutes** les migrations de l'app (supprime ses tables) ⚠️ |

---

#### Cas avancé : migration de données (remplir la table au démarrage)

Une migration peut aussi **insérer des lignes**, par exemple les thèmes de base. On crée une migration vide, puis on y écrit une fonction Python :

```bash
python manage.py makemigrations core --empty --name seed_topics
```

```python
from django.db import migrations


def create_topics(apps, schema_editor):
    # Toujours récupérer le modèle via apps.get_model(), pas via "from models import Topic"
    # (le modèle doit être celui tel qu'il était À CE MOMENT de l'historique)
    Topic = apps.get_model("core", "Topic")
    for name in ["debug", "pointers", "string manipulation", "algo", "memory"]:
        Topic.objects.get_or_create(name=name)


class Migration(migrations.Migration):
    dependencies = [("core", "0002_topic")]       # doit venir APRÈS la création de la table Topic
    operations = [
        migrations.RunPython(create_topics, migrations.RunPython.noop),
        # 1er argument : ce qu'on fait ; 2e : comment annuler (noop = ne rien faire)
    ]
```

> Pour des données plus volumineuses (des dizaines d'exercices avec leurs tests), une **commande perso** (`python manage.py seed_exercises`, voir la fin de cette section 3) est plus pratique : on peut la relancer à volonté.

---

#### Travailler à plusieurs avec les migrations

- ✅ **Toujours commit** les fichiers de migration avec `models.py` (dans le même commit).
- ✅ Après un `git pull` : lancer `python manage.py migrate` pour rattraper les migrations des autres.
- ❌ Ne **jamais modifier ni supprimer** une migration déjà commit/appliquée par quelqu'un d'autre.
- ⚠️ **Conflit classique** : deux personnes créent chacune une `0003_...` en parallèle. Git ne voit pas de conflit (fichiers de noms différents) mais Django si : `Conflicting migrations detected`. Solution :

```bash
python manage.py makemigrations --merge
```

- Pour limiter ce risque : **une seule personne** modifie `models.py` à la fois, ou on prévient l'équipe avant de toucher aux modèles.

**Tout recommencer de zéro** (en développement, si les données ne comptent pas et que tout est cassé) :

```bash
# 1. Vider la base (supprimer et recréer la base transcendence dans PostgreSQL)
# 2. Supprimer les fichiers 000X_*.py dans core/migrations/ (GARDER __init__.py)
# 3. Refaire :
python manage.py makemigrations
python manage.py migrate
```

⚠️ À ne **jamais** faire une fois qu'il y a de vraies données à conserver, ni sans prévenir l'équipe (tout le monde devrait alors faire pareil).

---

#### Erreurs courantes avec les migrations

| Message | Cause | Solution |
|---|---|---|
| `No changes detected` | `'core'` absent de `INSTALLED_APPS`, ou rien n'a changé | Vérifier `settings.py` |
| `You have 1 unapplied migration(s)` | `makemigrations` fait, mais pas `migrate` | `python manage.py migrate` |
| `relation "xxx" already exists` | La table existe déjà en base, mais Django ne le sait pas (migration non enregistrée) | En dev : recréer la base proprement |
| `Conflicting migrations detected` | Deux `000X` créées en parallèle | `makemigrations --merge` |
| `connection refused` / `could not connect to server` | PostgreSQL n'est pas lancé, ou mauvais `HOST`/`PORT` | Lancer PostgreSQL, vérifier `DATABASES` (dans Docker : `HOST` = nom du service) |
| `password authentication failed` | Mauvais `USER` / `PASSWORD` dans `DATABASES` | Vérifier les variables d'environnement |
| `database "transcendence" does not exist` | La base n'a pas été créée | `CREATE DATABASE transcendence;` dans `psql` |

---

### `core/admin.py` 🟡

**À quoi il sert**
**Enregistre les modèles dans l'interface d'administration web** (`/admin`). Une fois enregistré, un modèle peut être consulté, créé, modifié, supprimé depuis le navigateur, sans écrire de SQL.

**Ce qu'on peut faire dedans**

*Enregistrement minimal :*

```python
from django.contrib import admin
from .models import User, Exercise, Game

admin.site.register(User)
admin.site.register(Exercise)
```

*Enregistrement personnalisé (plus utile) avec `ModelAdmin` :*

```python
@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display  = ("username", "email", "elo", "in_game")   # colonnes de la liste
    search_fields = ("username", "email")                      # barre de recherche
    list_filter   = ("in_game",)                               # filtres sur le côté
    ordering      = ("-elo",)                                  # tri
    readonly_fields = ("password",)                            # champ non modifiable

@admin.register(Exercise)
class ExerciseAdmin(admin.ModelAdmin):
    list_display = ("nom", "elo", "debug", "pointers", "algo")
    search_fields = ("nom", "description")
```

*Afficher les tests directement dans la page d'un exercice (`Inline`) :*

```python
class TestInline(admin.TabularInline):
    model = Test
    extra = 1

@admin.register(Exercise)
class ExerciseAdmin(admin.ModelAdmin):
    inlines = [TestInline]
```

**Pour l'utiliser** :

```bash
python manage.py createsuperuser
python manage.py runserver        # puis http://localhost:8000/admin
```

**Utile pour notre projet**
- Très pratique pour **créer et corriger les exercices et leurs tests** à la main (sans écrire de route pour ça) et pour **déboguer** (voir les `Game`, les ELO...).
- ⚠️ Avec un `User` basé sur `AbstractUser`, enregistrer le modèle avec `UserAdmin` (`from django.contrib.auth.admin import UserAdmin`) pour que le mot de passe soit haché à la création depuis l'admin. Le `readonly_fields = ("password",)` ci-dessus est un pis-aller.
- ⚠️ Ne pas exposer `/admin` en production.

---

### `core/apps.py` 🟡

**À quoi il sert**
**Configuration de l'application** `core` (nom, type de clé primaire par défaut, code à exécuter au démarrage). Django la charge automatiquement via `INSTALLED_APPS`.

**Contenu par défaut**

```python
from django.apps import AppConfig

class CoreConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'core'
```

| Attribut | Rôle |
|---|---|
| `name` | Nom de l'app, **doit correspondre au dossier** (`'core'`) |
| `default_auto_field` | Type de clé primaire auto (`BigAutoField` = entier 64 bits) |
| `verbose_name` *(optionnel)* | Nom lisible dans l'admin |

**Ce qu'on peut faire dedans**
La seule chose vraiment utile est la méthode `ready()`, exécutée **une fois au démarrage**, pour brancher des *signals* (réactions automatiques à un événement de la base) :

```python
class CoreConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'core'

    def ready(self):
        from . import signals    # importe le fichier signals.py pour activer les signals
```

```python
# core/signals.py (à créer)
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import User

@receiver(post_save, sender=User)
def on_user_created(sender, instance, created, **kwargs):
    if created:
        print(f"Nouvel utilisateur : {instance.username}")   # ex: lui donner un 1er succès
```

**Utile pour notre projet**
- On n'y touche presque jamais.
- ℹ️ Le nom de la classe est `CoreConfig` (Django met en CamelCase le nom de l'app : `core` → `CoreConfig`).
- Les signals sont **optionnels** et parfois difficiles à déboguer : on peut tout aussi bien appeler la logique (attribuer un succès, etc.) depuis `core/services.py`.

---


### `core/views.py` 🔴 · et les fichiers de `api/` 🟢

**`core/views.py`** : le fichier généré par Django pour des vues classiques (pages HTML). **Non utilisé** : notre frontend est séparé et nos routes JSON vivent dans `api/` (voir section 4.4). On peut le laisser vide.

**`api/*.py`** : un fichier par groupe de routes de `routes.md`. Chaque route suit toujours le même schéma : *vérifier l'identité → valider les données (schema) → appeler un service de `core/services.py` → renvoyer un schema*. Détails et exemples en section 4.

---


### `core/tests.py` 🟡

**À quoi il sert**
Contient les **tests automatiques** de l'application. Quand on lance `python manage.py test`, Django crée une **base de test temporaire** (vide), exécute les tests, puis la supprime : la vraie base n'est jamais touchée.

> Le nom officiel est `tests.py` (avec un **s**). Il est aussi possible de le remplacer par un dossier `tests/`.

**Ce qu'on peut faire dedans**

```python
from django.test import TestCase
from .models import User, Friend

class UserModelTest(TestCase):
    def setUp(self):                       # exécuté avant CHAQUE test
        self.alice = User.objects.create(username="alice", email="a@a.com")
        self.bob   = User.objects.create(username="bob",   email="b@b.com")

    def test_default_elo(self):
        self.assertEqual(self.alice.elo, 1000)

    def test_win_probability_equal_players(self):
        self.assertAlmostEqual(self.alice.win_probability_against(self.bob), 0.5)

    def test_cannot_friend_twice(self):
        from django.db import IntegrityError
        Friend.objects.create(stalker=self.alice, followed=self.bob)
        with self.assertRaises(IntegrityError):
            Friend.objects.create(stalker=self.alice, followed=self.bob)
```

Assertions courantes : `assertEqual(a, b)`, `assertTrue(x)`, `assertFalse(x)`, `assertIn(a, liste)`, `assertRaises(Exception)`, `assertAlmostEqual(a, b)`.

Lancement :

```bash
python manage.py test                       # tous les tests
python manage.py test core.tests      # un fichier
python manage.py test --keepdb              # garde la base de test (plus rapide)
```

**Utile pour notre projet**
- Idéal pour tester la **logique de calcul** : formule ELO, règles de victoire, unicité des amitiés, contraintes de la base.
- Nécessite que l'utilisateur PostgreSQL ait le droit de **créer une base** (`CREATEDB`), sinon le test échoue au démarrage.
- Les routes de l'API se testent aussi ici, avec le client de test de Django (`self.client`), qui garde les cookies de session :

```python
import json
from django.test import TestCase

class AuthApiTest(TestCase):
    def test_sign_up_then_me(self):
        r = self.client.post("/api/auth/sign_up", json.dumps(
            {"username": "alice", "email": "a@a.com", "password": "S3cret!pass"}), content_type="application/json")
        self.assertEqual(r.status_code, 200)
        self.assertEqual(self.client.get("/api/users/me").status_code, 200)   # connecté grâce au cookie de session

    def test_me_requires_login(self):
        self.assertEqual(self.client.get("/api/users/me").status_code, 401)
```

- Les routes qui appellent Judge0 se testent en **simulant** `api.judge0.run` (`unittest.mock.patch`) : on ne lance pas de vrai Judge0 dans les tests.

---

### Fichiers optionnels qu'on pourrait ajouter

| Fichier | Rôle |
|---|---|
| `core/signals.py` | Réactions automatiques à un événement DB (voir `apps.py`) |
| `core/management/commands/seed_exercises.py` | Commande perso `python manage.py seed_exercises` pour remplir la base |
| `core/services.py` | Fonctions métier (calcul ELO, fin de match...) appelées par `api/` et `realtime/` |
| `.env` | Variables d'environnement (⚠️ dans `.gitignore`) |

Exemple de commande personnalisée :

```python
# core/management/commands/seed_exercises.py
from django.core.management.base import BaseCommand
from core.models import Exercise

class Command(BaseCommand):
    help = "Remplit la table Exercise avec des exercices de base"

    def handle(self, *args, **options):
        Exercise.objects.get_or_create(nom="ft_strlen", defaults={"description": "..."})
        self.stdout.write(self.style.SUCCESS("Exercices créés"))
```

---


## 4. Django comme backend complet (HTTP, WebSocket, Judge0)

### 4.1 Le principe

```
                       ┌─ HTTP  /api/...  ──▶ api/  (Django Ninja) ──┐
Navigateur ──HTTPS──▶  │                                              ├──▶ core/services.py ──▶ ORM ──▶ PostgreSQL
 (via reverse proxy)   └─ WebSocket /ws/ ──▶ realtime/ (Channels) ───┘          ▲
                                                   │                             │
                                                   └── Redis (messages entre joueurs)
                       api/ ──▶ httpx (async) ──▶ Judge0 (compilation, tests)
                       api/ ──▶ httpx (async) ──▶ LLM / RAG
```

Un **seul** processus Django (serveur ASGI `daphne`) répond à tout. Chaque couche a un rôle :

| Couche | Dossier | Rôle | Ce qu'elle ne fait **pas** |
|---|---|---|---|
| Routes HTTP | `api/` | Reçoit la requête, vérifie les données et l'identité, appelle un service, répond en JSON | Pas de calcul métier, pas de requête ORM compliquée |
| Temps réel | `realtime/` | Garde la connexion WebSocket ouverte, pousse les événements (`match.started`...) | Pas de logique de jeu |
| Logique métier | `core/services.py` | ELO, création de match, fin de match, amis, succès, transactions | Ne connaît ni HTTP ni WebSocket |
| Données | `core/models.py` | Tables et relations | |

### 4.2 Pourquoi les appels lents ne bloquent plus (le point qui motivait FastAPI)

Judge0 (compiler et tester du code) et le LLM peuvent prendre plusieurs secondes. Le danger : pendant ce temps, le serveur ne répond plus aux autres joueurs. La solution : **une vue `async` qui attend la réponse avec `await`**. Pendant l'attente, le serveur s'occupe d'autres requêtes.

```python
async def run_code(...):
    result = await judge0.run(...)       # ✅ le serveur est libre pendant l'attente
    # result = requests.post(...)        # ❌ requests est synchrone : il bloquerait le serveur
```

Règles qui découlent de ça :

| # | Règle | Pourquoi |
|---|---|---|
| 1 | **Les endpoints sont des `async def`.** | Sous ASGI, une vue `def` est exécutée dans un thread partagé (un à la fois) : une vue lente ralentirait les autres. |
| 2 | **Appels réseau (Judge0, LLM) avec `httpx.AsyncClient`**, jamais `requests`. | `requests` bloque tout le processus. |
| 3 | **Lectures/écritures simples de l'ORM : méthodes asynchrones** `aget`, `afirst`, `acreate`, `asave`, `adelete`, `acount`, `aexists`, et `async for` pour parcourir un résultat. | L'ORM classique dans une `async def` lève `SynchronousOnlyOperation`. |
| 4 | **Logique avec transaction (`transaction.atomic()`, `select_for_update()`) : une fonction synchrone de `services.py`, appelée avec `sync_to_async`** (ou `database_sync_to_async` dans un consumer). | `transaction.atomic()` ne fonctionne pas directement dans du code `async`. |
| 5 | **Ne jamais attendre Judge0 à l'intérieur d'une transaction.** D'abord Judge0, ensuite la transaction courte qui enregistre le résultat. | Une transaction ouverte pendant plusieurs secondes garde des verrous et bloque les autres joueurs. |

```python
from asgiref.sync import sync_to_async

# Lecture simple : ORM async
user = await User.objects.filter(username=username).afirst()

# Résultat en liste : "async for" ou compréhension asynchrone
top = [u async for u in User.objects.order_by("-elo").values("username", "elo")[:50]]

# Logique avec transaction : service synchrone + sync_to_async
result = await sync_to_async(services.finish_match)(game_id, winner_id)
```

> ℹ️ Les méthodes `a...` de l'ORM existent depuis Django 4.1 et sont en pratique exécutées dans un thread par Django (la base reste synchrone en dessous). Ce n'est pas un problème : ce qui compte, c'est que **l'attente de Judge0/LLM** soit réellement asynchrone.

### 4.3 `config/asgi.py` : servir HTTP et WebSocket

```python
# config/asgi.py
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
django_asgi_app = get_asgi_application()          # AVANT d'importer realtime (initialise Django)

from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.security.websocket import AllowedHostsOriginValidator
import realtime.routing

application = ProtocolTypeRouter({
    "http": django_asgi_app,                                      # /api/..., /admin
    "websocket": AllowedHostsOriginValidator(                     # refuse les WebSocket venant d'un autre site
        AuthMiddlewareStack(                                      # identifie le joueur grâce à son cookie de session
            URLRouter(realtime.routing.websocket_urlpatterns)
        )
    ),
})
```

Lancement :

```bash
python manage.py runserver                              # dev (grâce à 'daphne' dans INSTALLED_APPS)
daphne -b 0.0.0.0 -p 8000 config.asgi:application       # production / Docker
```

### 4.4 Les routes HTTP avec Django Ninja

**Pourquoi Ninja ?** C'est le plus proche de ce que l'équipe prévoyait avec FastAPI : décorateurs `@router.get(...)`, validation automatique des données avec des *schemas* (Pydantic), documentation interactive générée sur `/api/docs`, et support natif des vues `async`.

**L'objet principal** :

```python
# api/api.py
from ninja import NinjaAPI
from . import auth, users, friends, leaderboard, catalog, learn, matches

api = NinjaAPI(title="Transcendence API")

api.add_router("/auth/", auth.router)
api.add_router("/users/", users.router)
api.add_router("/friends/", friends.router)
api.add_router("/leaderboard/", leaderboard.router)
api.add_router("/", catalog.router)           # /topics, /exercises/:id
api.add_router("/learn/", learn.router)
api.add_router("/matches/", matches.router)
```

**Les schémas** (forme des données, validées automatiquement ; une donnée invalide → erreur 422 sans écrire une ligne) :

```python
# api/schemas.py
from ninja import Schema

class SignUpIn(Schema):
    username: str
    email: str
    password: str

class LoginIn(Schema):
    username: str
    password: str

class SubmitIn(Schema):                   # POST /api/matches/:id/submit
    source: str

class UserPublicOut(Schema):
    username: str
    elo: int
    level_solo: int
    github: str = ""

class InvitationIn(Schema):               # POST /api/matches/invitations
    opponent_id: int
    topic: str
    ranked: bool = True
```

**Les routes** (un exemple de lecture, un d'écriture) :

```python
# api/users.py
from ninja import Router
from ninja.errors import HttpError
from ninja.security import django_auth
from core.models import User
from .schemas import UserPublicOut

router = Router()

@router.get("/me", auth=django_auth, response=UserPublicOut)       # GET /api/users/me
async def me(request):
    return request.auth                            # l'utilisateur connecté (grâce au cookie de session)

@router.get("/", response=list[UserPublicOut], auth=django_auth)   # GET /api/users?search=eli
async def search(request, search: str = ""):
    qs = User.objects.filter(username__icontains=search)[:20]
    return [u async for u in qs]

@router.get("/{username}", response=UserPublicOut, auth=django_auth)    # GET /api/users/alice
async def profile(request, username: str):
    user = await User.objects.filter(username=username).afirst()
    if user is None:
        raise HttpError(404, "User not found")
    return user
```

- `auth=django_auth` : la route exige un joueur connecté (session) ; sinon réponse 401. Pour les écritures (POST/PUT/DELETE), il vérifie aussi le **jeton CSRF** : le frontend doit renvoyer le cookie `csrftoken` dans l'en-tête `X-CSRFToken`.
- Le type de retour (`response=...`) filtre ce qui sort : **le mot de passe (ou l'email des autres joueurs) ne peut pas fuiter par accident**, il suffit de ne pas le mettre dans le schéma.
- ⚠️ L'ordre compte : `/me` doit être déclaré **avant** `/{username}`, sinon `me` serait pris pour un pseudo.

> **Alternative : Django REST Framework (DRF).** Plus répandu et très documenté, mais ses vues sont **synchrones** par défaut : pour Judge0 il faudrait ajouter une extension (`adrf`) ou séparer. Si l'équipe préfère DRF, le reste de la doc (modèles, services, Channels, Docker) ne change pas ; seuls les fichiers de `api/` sont réécrits.

### 4.5 L'authentification (`/api/auth/*`)

On utilise les **sessions de Django** : après login, Django pose un cookie ; chaque requête suivante (HTTP **et** WebSocket) est automatiquement identifiée par ce cookie.

```python
# api/auth.py
from django.contrib.auth import aauthenticate, alogin, alogout
from django.db import IntegrityError
from ninja import Router
from ninja.errors import HttpError
from ninja.security import django_auth
from core.models import User
from .schemas import SignUpIn, LoginIn

router = Router()

@router.post("/sign_up")                                          # POST /api/auth/sign_up
async def sign_up(request, data: SignUpIn):
    try:
        user = await User.objects.acreate_user(                  # mot de passe haché et salé
            username=data.username, email=data.email, password=data.password)
    except IntegrityError:
        raise HttpError(409, "Username or email already used")
    await alogin(request, user)
    return {"id": user.id}

@router.put("/login")                                             # PUT /api/auth/login (voir routes.md)
async def login(request, data: LoginIn):
    user = await aauthenticate(request, username=data.username, password=data.password)
    if user is None:
        raise HttpError(401, "Invalid credentials")
    await alogin(request, user)
    return {"id": user.id}

@router.post("/logout", auth=django_auth)                         # POST /api/auth/logout
async def logout(request):
    await alogout(request)
    return {"ok": True}
```

Points d'attention :
- Un `PUT` pour le login est inhabituel (la convention est `POST /api/auth/login`). Ça marche, mais autant corriger `routes.md` pour éviter la confusion.
- Valider le mot de passe à l'inscription avec `django.contrib.auth.password_validation.validate_password` (les règles de `AUTH_PASSWORD_VALIDATORS`).
- La session évite d'écrire à la main un système de tokens. Si l'équipe veut un jour de la 2FA ou de l'OAuth (par exemple « se connecter avec 42 »), la bibliothèque `django-allauth` se branche sur ce même modèle `User`.
- Les méthodes `aauthenticate`, `alogin`, `alogout` et `acreate_user` existent dans les versions récentes de Django (5.x). Si une version plus ancienne est utilisée, les remplacer par leurs équivalents synchrones avec `sync_to_async`.

### 4.6 Judge0 : `run` et `submit` sans bloquer le serveur

Les routes `POST /api/exercises/:id/run`, `.../submit` et `POST /api/matches/:id/run|submit` envoient du code à Judge0 et **attendent** le résultat. Le client est isolé dans `api/judge0.py` :

```python
# api/judge0.py
import httpx
from django.conf import settings

C_LANGUAGE_ID = 50      # "C (GCC)" dans les versions courantes de Judge0 : à vérifier avec GET {JUDGE0_URL}/languages

async def run(source_code: str, stdin: str = "", expected_out: str | None = None) -> dict:
    payload = {"source_code": source_code, "language_id": C_LANGUAGE_ID, "stdin": stdin}
    if expected_out is not None:
        payload["expected_output"] = expected_out
    async with httpx.AsyncClient(timeout=30) as client:
        r = await client.post(
            f"{settings.JUDGE0_URL}/submissions",
            params={"base64_encoded": "false", "wait": "true"},    # wait=true : Judge0 répond quand l'exécution est finie
            json=payload,
        )
    r.raise_for_status()
    return r.json()      # stdout, stderr, compile_output, status {id, description}, time, memory...
```

Exemple de route `submit` d'un match, en respectant la règle « Judge0 d'abord, transaction ensuite » :

```python
# api/matches.py (extrait)
from asgiref.sync import sync_to_async
from ninja.security import django_auth
from core import services
from . import judge0

@router.post("/{match_id}/submit", auth=django_auth)               # POST /api/matches/:id/submit
async def submit(request, match_id: int, code: SubmitIn):
    user = request.auth
    ctx = await sync_to_async(services.get_submit_context)(match_id, user.id)   # vérifie : match en cours, c'est bien mon match, temps restant
    verdicts = []
    for test in ctx["tests"]:                                                    # tests de l'exercice (core.models.Test)
        out = await judge0.run(code.source, stdin=test["input"], expected_out=test["expected_out"])
        verdicts.append(out)
    # transaction courte : on enregistre la tentative, on retire les points, on prévient l'adversaire
    result = await sync_to_async(services.record_submission)(match_id, user.id, code.source, verdicts)
    return result       # sortie du code ou erreur (segfault, ne compile pas, 50% de réussite, fonction interdite...)
```

Points d'attention :
- **Le serveur est la seule source de vérité.** Les `tests` (et surtout `reference_solution`, `test_code`) ne sortent **jamais** vers le navigateur : un schéma de réponse ne contient que ce que le joueur a le droit de voir.
- **Fonctions interdites** (`Exercise.authorized`) : à vérifier **avant** d'envoyer à Judge0 (analyse du code source) et/ou via le `test_code`.
- Judge0 doit être **isolé** (le code des joueurs est du code non fiable) : il tourne dans ses propres conteneurs, jamais dans le conteneur Django.
- L'attente `wait=true` est simple, mais si Judge0 devient lent, passer à un mode sans attente (`GET /submissions/{token}` en boucle avec `await asyncio.sleep(...)`).
- Le **temps** compte dans le jeu (`time1`, `time2`) : le mesurer côté serveur (heure de la soumission − `started_at`), jamais avec une valeur envoyée par le navigateur.

### 4.7 Le temps réel avec Channels (`/ws/`)

Les événements de `routes.md` (partie 3 « Course of a game ») sont poussés vers le navigateur par un **seul WebSocket par joueur**. À la connexion, le joueur est placé dans un « groupe » à son nom ; pour lui envoyer un message, on écrit dans ce groupe.

```python
# realtime/routing.py
from django.urls import path
from . import consumers

websocket_urlpatterns = [path("ws/", consumers.UserConsumer.as_asgi())]
```

```python
# realtime/consumers.py
from channels.db import database_sync_to_async
from channels.generic.websocket import AsyncJsonWebsocketConsumer
from core import services

class UserConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        user = self.scope["user"]                       # rempli par AuthMiddlewareStack (cookie de session)
        if not user.is_authenticated:
            await self.close()
            return
        self.group = f"user_{user.id}"
        await self.channel_layer.group_add(self.group, self.channel_name)
        await self.accept()
        friend_ids = await database_sync_to_async(services.set_online)(user.id, True)
        for fid in friend_ids:
            await notify(fid, "friend.online", {"user_id": user.id})

    async def disconnect(self, code):
        user = self.scope["user"]
        if user.is_authenticated:
            await self.channel_layer.group_discard(self.group, self.channel_name)
            friend_ids = await database_sync_to_async(services.set_online)(user.id, False)
            for fid in friend_ids:
                await notify(fid, "friend.offline", {"user_id": user.id})

    async def event(self, message):                     # appelé quand on écrit dans le groupe (voir notify)
        await self.send_json({"event": message["name"], "data": message["data"]})
```

```python
# realtime/events.py : à appeler de n'importe où (route, service...)
from channels.layers import get_channel_layer

async def notify(user_id: int, name: str, data: dict):
    await get_channel_layer().group_send(
        f"user_{user_id}",
        {"type": "event", "name": name, "data": data},      # "type": "event" → appelle la méthode event() du consumer
    )
```

Dans un service **synchrone**, on utilise `async_to_sync(notify)(...)`. Les événements de `routes.md` se déclenchent aux endroits suivants :

| Événement | Déclenché par | Envoyé à |
|---|---|---|
| `friend.online` / `friend.offline` | connexion / déconnexion du WebSocket | les amis |
| `friend.request.received` / `.accepted` | `PUT /users/:username/add_friend` et acceptation | le destinataire / l'émetteur |
| `invitation.received` | `POST /matches/invitations` | l'adversaire |
| `invitation.accepted` / `declined` | `POST /invitations/:id/accept` ou `/decline` | l'émetteur |
| `invitation.expired` | expiration (voir 4.8) | les deux |
| `match.ready` | `POST /matches/:id/ready` | l'autre joueur |
| `match.started` `{ends_at}` | quand les deux sont prêts | les deux |
| `opponent.submitted` | `POST /matches/:id/submit` | l'adversaire (**sans verdict ni score**) |
| `match.finished` `{result, score, elo_delta}` | fin de match, forfait ou temps écoulé | les deux |
| `achievement.unlocked` | un service débloque un succès | le joueur |

**Les WebSockets ne servent qu'à *prévenir*.** Les actions du joueur (prêt, soumettre, abandonner) passent par les routes HTTP de `routes.md` ; au besoin, le navigateur rafraîchit son état avec `GET /api/matches/:id` ou `GET /api/matches/current` (par exemple après une reconnexion). Un événement perdu ne doit jamais casser le jeu.

### 4.8 Correspondance `routes.md` ↔ code

| Groupe de routes | Fichier | Fonctions de `core/services.py` (à écrire) | Remarques |
|---|---|---|---|
| `/api/auth/*` | `api/auth.py` | | Voir 4.5 |
| `/api/users/*` | `api/users.py` | `get_profile_stats`, `list_matches`, `list_achievements` | `/me/avatar` : reçoit un fichier (`UploadedFile` de Ninja), vérifier **type et taille** |
| `/api/friends/*` | `api/friends.py` | `list_friends`, `remove_friend`, `set_online` | `add_friend` est sous `/users/:username/` |
| `/api/leaderboard/*` | `api/leaderboard.py` | `get_leaderboard_users`, `get_leaderboard_exercises` | Voir les requêtes `aggregate`/`annotate` (partie 8.6) |
| `/api/topics`, `/api/exercises/:id` | `api/catalog.py` | | **Ne jamais renvoyer** `tests`, `reference_solution` |
| `/api/learn/*` | `api/learn.py` | `get_learn_path`, `get_chapter` | « locked / in progress / done » calculé depuis `SoloProgress` |
| `/api/exercises/:id/run\|submit` | `api/learn.py` | `record_solo_submission` | Judge0, sans ELO |
| `/api/matches/invitations*` | `api/matches.py` | `create_invitation`, `accept_invitation`, `decline_invitation` | `accept` crée le `Game` et choisit l'exercice (selon `topic`) dans **une** transaction |
| `/api/matches/current`, `/:id` | `api/matches.py` | `get_current_match`, `get_match` | Ne pas révéler l'avancement de l'adversaire |
| `/api/matches/:id/ready\|run\|submit\|forfeit\|result` | `api/matches.py` | `set_ready`, `record_submission`, `forfeit_match`, `get_result` | `finish_match` (partie 9) est appelé à la fin, en forfait ou à `ends_at` |

**Routes qui semblent manquer dans `routes.md`** (à discuter en équipe) :

- accepter / refuser une demande d'ami : par exemple `POST /api/friends/requests/:id/accept` et `.../decline` (les événements `friend.request.*` existent mais pas la route) ;
- modifier ses infos et son mot de passe, supprimer son compte (la page `/settings` le prévoit) : `PATCH /api/users/me`, `PUT /api/users/me/password`, `DELETE /api/users/me` ;
- un moyen de savoir si on est connecté au chargement de la page (`GET /api/users/me` répondant 401 convient) ;
- RGPD/suppression de compte : prévoir `on_delete=SET_NULL` sur `Game` pour garder l'historique des adversaires (voir section 5) ;
- le LLM / RAG : aucune route n'apparaît encore (par exemple `POST /api/exercises/:id/hint`, avec une réponse en *streaming*).

**Les délais (invitation, match) sans tâche de fond.** Chaque ligne a un horodatage (`expires_at` d'une invitation, `ends_at` d'un match). Au lieu de programmer un minuteur qui peut se perdre au redémarrage, on **vérifie à chaque lecture** : si `now > ends_at` et que le match est encore `playing`, le service appelle `finish_match` (qui doit être **idempotent** : le rappeler sur un match terminé ne fait rien). Pour pousser `match.finished` à l'heure exacte, un `asyncio.create_task` ou une tâche périodique peut s'ajouter plus tard, mais la base reste l'unique source de vérité.

### 4.9 Pièges à connaître

- **`SynchronousOnlyOperation`** : ORM classique (`.get()`, `.save()`, boucle `for` sur un QuerySet) dans une fonction `async def`. Utiliser les méthodes `a...`, `async for`, ou `sync_to_async`.
- **Ne renvoyer que des données déjà lues** (liste, dictionnaire, objet), jamais un `QuerySet` non évalué dans une réponse. Un `QuerySet` est « paresseux » (partie 8.7).
- **Accéder à une relation dans du code async** (`game.exercise.nom`) déclenche une requête synchrone cachée → erreur. Charger la relation à l'avance avec `select_related` / `prefetch_related` (partie 8.8), ou la lire avec `await Game.objects.select_related("exercise").aget(...)`.
- **`request.user` en async** : utiliser `await request.auser()` ou `request.auth`, pas `request.user.xxx` directement. Si `django_auth` pose problème dans une vue `async`, écrire une petite classe d'authentification asynchrone (`class AsyncSessionAuth(APIKeyCookie)`...) : à tester dès les premières routes.
- **Connexions à PostgreSQL** : pour les requêtes HTTP, Django ouvre/ferme les connexions automatiquement. Dans les **consumers**, utiliser `database_sync_to_async` (et non `sync_to_async`) : il referme les connexions périmées. Sans ça, on finit par voir `too many clients already`.
- **Les transactions** (`transaction.atomic()`, `select_for_update()`) restent à l'intérieur d'**une** fonction de `services.py`, jamais à cheval sur plusieurs appels (voir la Partie 9 de la section `models.py`).
- **Condition de course sur les actions critiques** : deux joueurs qui cliquent « prêt » ou deux `submit` simultanés → verrouiller la ligne `Game` avec `select_for_update()`. Le sujet exige l'absence de race condition.
- **Ne jamais faire confiance au navigateur** : le score, le temps et l'ELO sont toujours calculés côté serveur.

### 4.10 Migrations et Docker

`manage.py` reste le point d'entrée des migrations. Dans Docker, on applique les migrations au démarrage du conteneur backend, une fois PostgreSQL prêt. Extrait indicatif (à adapter : ports, reverse proxy HTTPS, Judge0 qui a ses propres services, frontend) :

```yaml
services:
  db:
    image: postgres:16
    env_file: .env                     # POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 10

  backend:
    build: .
    env_file: .env                     # avec POSTGRES_HOST=db, REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy     # attend que PostgreSQL réponde
      redis:
        condition: service_healthy
    volumes:
      - media:/app/media               # avatars conservés entre les redémarrages
    command: sh -c "python manage.py migrate && daphne -b 0.0.0.0 -p 8000 config.asgi:application"

volumes:
  pgdata:
  media:
```

- Judge0 se lance avec **ses propres conteneurs** (serveur, workers, sa propre base et son Redis ; voir sa documentation) ; Django n'en connaît que l'URL (`JUDGE0_URL`).
- Le **reverse proxy HTTPS** (nginx, Caddy...) envoie `/api/`, `/admin/`, `/media/` **et `/ws/`** vers `backend:8000`. Pour `/ws/`, il doit laisser passer l'upgrade WebSocket (en-têtes `Upgrade` et `Connection`) ; c'est l'oubli classique quand « ça marche en local mais pas dans Docker ».
- Pour la production : `DEBUG=0`, `SECRET_KEY` unique, collecter les fichiers statiques de l'admin (`collectstatic`) et ne pas exposer `/admin` publiquement.

⚠️ L'image `postgres` ne crée la base (`POSTGRES_DB`) qu'au **tout premier démarrage**, quand le volume est vide. Si on change ces variables ensuite, il faut supprimer le volume (`docker compose down -v`) pour qu'elles soient prises en compte.

---

## 5. Correspondance avec la DB

### 5.1 Les tables du tableau blanc

| Table (tableau) | Champs notés sur le tableau | Remarque |
|---|---|---|
| **User** | id, username, email, password, github, elo, total_score, level_solo, in_game | `username`, `email`, `password` fournis par `AbstractUser` |
| **Exercise** | id, nom, program, prototype, description, hints, authorized, pre_body, elo, debug, pointers, str_man, algo, f_rep | Les 5 booléens de catégorie deviennent une table `Topic` (voir ci-dessous) |
| **Test** | id, exercise (EXID), compiler_code, expected_out, input, test_code, reference_solution | |
| **Games** | id, exercise (EX ID), challenger, victim, score1, score2, attempts1, attempts2, time1, time2 | Modèle `Game` (singulier) |
| **Achievements** | id, user (USER ID), A1…A5 | Voir la proposition ci-dessous |
| **Friends** | id, stalker, followed | Il manque l'état de la demande (en attente / acceptée) |

### 5.2 Ce que `routes.md` demande en plus du tableau

Les routes et les événements temps réel utilisent des notions qui n'ont pas encore de colonne :

| Besoin (dans `routes.md`) | Ajout proposé |
|---|---|
| Demandes d'amis (`friend.request.received / accepted`) | `Friend.status` (`pending` / `accepted`) |
| Invitations (`/api/matches/invitations`, `expired`) | table `Invitation` (émetteur, adversaire, thème, `ranked`, statut, `expires_at`) |
| Match : lobby, prêt, `ends_at`, forfait, résultat, ELO deltas | `Game.status`, `ranked`, `ready1/2`, `started_at`, `ends_at`, `winner`, `forfeited_by`, `elo_delta1/2` |
| Thèmes (`/api/topics`, choix du sujet de l'invitation) | table `Topic` + `Exercise.topics` (N ↔ N), à la place des 5 booléens |
| Parcours d'apprentissage (`/learn`, chapitres verrouillés / en cours / finis) | table `Chapter` + `Exercise.chapter` ; table `SoloProgress` |
| Classement des exercices (taux de réussite, meilleur temps) | table `Submission` (une ligne par soumission) |
| Avatar (`PUT /api/users/me/avatar`) | `User.avatar` |
| « Achievements » avec `A1…A5` en colonnes | table `Achievement` (catalogue) + `UserAchievement` (qui a débloqué quoi). Avec `A1…A5` en colonnes, ajouter un succès obligerait à modifier la table et chaque joueur aurait des colonnes à vide. |

> ⚠️ Le tableau indique quelques ✓ / ✗ à côté des catégories d'exercice (`debug ✗`, `pointers ✓`, `str_man ✗`, `algo ✗`, `f_rep ✓`). Si ce sont les valeurs d'un exercice d'exemple, elles se traduisent par `exercice.topics = [pointers, f_rep]` (le `.add(...)` de la partie 4.4). Les 5 thèmes sont créés par une migration de données (voir « Cas avancé » dans la section migrations).

### 5.3 `core/models.py` complet (proposition)

```python
from django.conf import settings
from django.contrib.auth.models import AbstractUser
from django.db import models
from django.db.models import F, Q


class User(AbstractUser):
    # AbstractUser fournit déjà : username, password (haché et salé), email, is_active, last_login, date_joined...
    email       = models.EmailField(unique=True)           # surcharge : l'email doit être unique
    github      = models.CharField(max_length=100, blank=True)
    avatar      = models.ImageField(upload_to="avatars/", blank=True)    # nécessite Pillow
    elo         = models.IntegerField(default=1000, db_index=True)       # index : classement
    total_score = models.IntegerField(default=0)
    level_solo  = models.IntegerField(default=0)
    in_game     = models.BooleanField(default=False)

    def __str__(self):
        return f"{self.username} ({self.elo})"

    def win_probability_against(self, other):
        """Formule ELO : probabilité que self batte other (entre 0 et 1)."""
        return 1 / (1 + 10 ** ((other.elo - self.elo) / 400))


class Topic(models.Model):
    """debug, pointers, string manipulation, algo, f_rep... (créés par une migration de données)"""
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name


class Chapter(models.Model):
    """Un chapitre du parcours d'apprentissage (/learn)."""
    title    = models.CharField(max_length=100)
    position = models.PositiveIntegerField(unique=True)     # ordre dans le parcours

    class Meta:
        ordering = ["position"]


class Exercise(models.Model):
    nom         = models.CharField(max_length=100, unique=True)
    program     = models.TextField()                         # sujet de l'exercice
    prototype   = models.TextField()
    description = models.TextField()
    hints       = models.JSONField(default=list, blank=True)
    authorized  = models.JSONField(default=list, blank=True) # fonctions autorisées (["write", "malloc"...])
    pre_body    = models.TextField(blank=True)               # code de départ affiché dans l'éditeur
    elo         = models.IntegerField(default=1000, db_index=True)
    topics      = models.ManyToManyField(Topic, related_name="exercises", blank=True)
    chapter     = models.ForeignKey(Chapter, null=True, blank=True, on_delete=models.SET_NULL, related_name="exercises")
    position    = models.PositiveIntegerField(default=0)     # ordre dans le chapitre

    class Meta:
        ordering = ["chapter", "position"]


class Test(models.Model):
    exercise           = models.ForeignKey(Exercise, on_delete=models.CASCADE, related_name="tests")
    compiler_code      = models.TextField()
    expected_out       = models.TextField()
    input              = models.TextField(blank=True)
    test_code          = models.TextField()
    reference_solution = models.TextField()                  # ⚠️ ne JAMAIS l'envoyer au navigateur


class SoloProgress(models.Model):
    """Où en est un joueur dans le parcours d'apprentissage (pas d'ELO)."""
    user         = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="solo_progress")
    exercise     = models.ForeignKey(Exercise, on_delete=models.CASCADE, related_name="solo_progress")
    completed_at = models.DateTimeField(null=True, blank=True)   # NULL = commencé mais pas fini

    class Meta:
        constraints = [models.UniqueConstraint(fields=["user", "exercise"], name="uniq_progress_user_exercise")]


class Friend(models.Model):
    class Status(models.TextChoices):
        PENDING  = "pending",  "En attente"
        ACCEPTED = "accepted", "Acceptée"

    stalker    = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="following")   # celui qui demande
    followed   = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="followers")   # celui qui reçoit
    status     = models.CharField(max_length=10, choices=Status.choices, default=Status.PENDING)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["stalker", "followed"], name="uniq_friend_pair"),            # pas de doublon
            models.CheckConstraint(condition=~Q(stalker=F("followed")), name="no_self_friend"),          # pas d'ami avec soi-même
        ]


class Game(models.Model):
    class Status(models.TextChoices):
        LOBBY     = "lobby",     "Salle d'attente"      # créé à l'acceptation de l'invitation
        PLAYING   = "playing",   "En cours"             # les deux sont prêts, ends_at défini
        FINISHED  = "finished",  "Terminé"
        CANCELLED = "cancelled", "Annulé"

    exercise     = models.ForeignKey(Exercise, on_delete=models.PROTECT, related_name="games")
    challenger   = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, on_delete=models.SET_NULL, related_name="games_as_challenger")
    victim       = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, on_delete=models.SET_NULL, related_name="games_as_victim")
    ranked       = models.BooleanField(default=True)
    status       = models.CharField(max_length=10, choices=Status.choices, default=Status.LOBBY, db_index=True)
    ready1       = models.BooleanField(default=False)
    ready2       = models.BooleanField(default=False)
    started_at   = models.DateTimeField(null=True, blank=True)
    ends_at      = models.DateTimeField(null=True, blank=True)      # fin du match (le serveur décide)
    score1       = models.IntegerField(default=0)
    score2       = models.IntegerField(default=0)
    attempts1    = models.IntegerField(default=0)
    attempts2    = models.IntegerField(default=0)
    time1        = models.IntegerField(null=True, blank=True)       # secondes (NULL tant que pas terminé)
    time2        = models.IntegerField(null=True, blank=True)
    winner       = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, blank=True, on_delete=models.SET_NULL, related_name="games_won")
    forfeited_by = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, blank=True, on_delete=models.SET_NULL, related_name="games_forfeited")
    elo_delta1   = models.IntegerField(null=True, blank=True)
    elo_delta2   = models.IntegerField(null=True, blank=True)
    created_at   = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            # un joueur ne peut être que dans un seul match en cours
            # (à compléter dans les services avec select_for_update, voir section 4.9)
            models.CheckConstraint(condition=~Q(challenger=F("victim")), name="no_self_game"),
        ]


class Invitation(models.Model):
    class Status(models.TextChoices):
        PENDING  = "pending",  "En attente"
        ACCEPTED = "accepted", "Acceptée"
        DECLINED = "declined", "Refusée"
        EXPIRED  = "expired",  "Expirée"

    sender     = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="invitations_sent")
    opponent   = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="invitations_received")
    topic      = models.ForeignKey(Topic, null=True, blank=True, on_delete=models.SET_NULL)    # NULL = aléatoire
    ranked     = models.BooleanField(default=True)
    status     = models.CharField(max_length=10, choices=Status.choices, default=Status.PENDING, db_index=True)
    game       = models.OneToOneField(Game, null=True, blank=True, on_delete=models.SET_NULL, related_name="invitation")
    created_at = models.DateTimeField(auto_now_add=True)
    expires_at = models.DateTimeField()


class Submission(models.Model):
    """Une soumission de code (solo ou match). Sert aux classements d'exercices et à l'historique."""
    user       = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="submissions")
    exercise   = models.ForeignKey(Exercise, on_delete=models.CASCADE, related_name="submissions")
    game       = models.ForeignKey(Game, null=True, blank=True, on_delete=models.SET_NULL, related_name="submissions")   # NULL = solo
    passed     = models.BooleanField(default=False)
    success_rate = models.FloatField(default=0)               # part des tests réussis (0 à 1)
    runtime_ms = models.IntegerField(null=True, blank=True)
    verdict    = models.CharField(max_length=30, blank=True)   # "ok", "compile_error", "segfault", "forbidden_function"...
    created_at = models.DateTimeField(auto_now_add=True)


class Achievement(models.Model):
    """Le catalogue des succès (A1, A2... du tableau)."""
    code        = models.SlugField(unique=True)               # "first_win", "ten_exercises"...
    title       = models.CharField(max_length=100)
    description = models.TextField(blank=True)


class UserAchievement(models.Model):
    user        = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="achievements")
    achievement = models.ForeignKey(Achievement, on_delete=models.CASCADE, related_name="unlocked_by")
    unlocked_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["user", "achievement"], name="uniq_user_achievement")]
```

**Les choix à connaître**

| Choix | Pourquoi |
|---|---|
| `settings.AUTH_USER_MODEL` (au lieu de `User`) dans les `ForeignKey` | Bonne pratique : on désigne « le modèle utilisateur du projet » sans l'importer. |
| `Game.challenger/victim` en `SET_NULL` (et `null=True`) | Si un joueur supprime son compte, l'historique de l'adversaire est conservé (au lieu de `CASCADE` qui effacerait le match des deux). |
| `Game.exercise` en `PROTECT` | Un exercice déjà joué ne peut pas être supprimé : l'historique casserait. |
| `UniqueConstraint` / `CheckConstraint` dans `Meta.constraints` | Remplace `unique_together` (jugé « legacy » par la documentation Django) et permet aussi des règles comme « pas d'ami avec soi-même ». Sur une version de Django antérieure à 5.1, `CheckConstraint` s'écrit avec `check=` au lieu de `condition=`. |
| `Friend` : une ligne par demande, `status` pour l'état | `alice → bob` pending = demande envoyée ; `accepted` = amis. Pour tester l'amitié dans les deux sens : `Friend.objects.filter(Q(stalker=a, followed=b) | Q(stalker=b, followed=a), status="accepted")`. |
| Pas de colonne `score_total` calculée à la main ailleurs | `total_score` est mis à jour dans les services avec `F("total_score") + n` (partie 8.5) pour éviter les pertes. |

**Requêtes typiques** (pour les routes de `routes.md`) :

```python
# GET /api/leaderboard/users
User.objects.order_by("-elo").values("username", "elo")[:50]

# GET /api/users/:username/matches  (historique)
Game.objects.filter(Q(challenger=user) | Q(victim=user), status=Game.Status.FINISHED) \
            .select_related("exercise", "challenger", "victim").order_by("-created_at")

# GET /api/leaderboard/exercises  (taux de réussite et meilleur temps)
from django.db.models import Avg, Min, Count
Exercise.objects.annotate(
    nb=Count("submissions"),
    success=Avg("submissions__success_rate"),
    best_ms=Min("submissions__runtime_ms", filter=Q(submissions__passed=True)),
).order_by("-success")

# GET /api/friends  (amis acceptés d'alice)
User.objects.filter(Q(followers__stalker=alice, followers__status="accepted") |
                    Q(following__followed=alice, following__status="accepted")).distinct()

# Exercice aléatoire d'un thème, pour accepter une invitation
Exercise.objects.filter(topics=topic).order_by("?").first()
```

> Les noms de champs (`nom`, `program`...) viennent du tableau ; si l'équipe préfère l'anglais partout (`name`, `statement`), c'est le moment de le décider, **avant la première migration** (renommer plus tard est possible mais crée des migrations en plus).

---

## 6. Lien avec le sujet

Ce que le sujet exige et qui concerne la base de données ou son accès :

| Exigence du sujet | Comment on y répond | Où dans cette doc |
|---|---|---|
| Identifiants dans un `.env` ignoré par git + un `.env.example` fourni | `python-dotenv`, variables d'environnement | Section 3, `settings.py`, point 1 |
| « The database must have a clear schema and well-defined relations » | Modèles avec `ForeignKey`, contraintes (`UniqueConstraint`, `CheckConstraint`), `on_delete` | Section 3, `models.py`, Parties 4 à 6 ; section 5 |
| Module Web **mineur** « Use an ORM for the database » (1 pt) | ORM Django | Toute la doc |
| Module Web « framework backend » (mineur ; majeur s'il est combiné à un framework frontend) | Django (+ Django Ninja) : à justifier dans le README (structure, ORM intégré, authentification, admin, écosystème). *L'ancienne version de la doc justifiait FastAPI : ce paragraphe est à réécrire dans le README.* | Section 4 |
| Temps réel (WebSockets), si le module est visé | Django Channels + Redis | Section 4.7 |
| Mots de passe hachés et salés | `create_user` / `make_password` (`AbstractUser`) | Section 3, `settings.py`, point 4 ; section 4.5 |
| Utilisateurs simultanés, **pas de corruption de données ni de race condition** | `transaction.atomic()`, `select_for_update()`, `F()`, contraintes d'unicité | Section 3, `models.py`, Parties 8.5 et 9 ; section 4.9 |
| Déploiement en **une seule commande** avec Docker | `docker compose up` (PostgreSQL + Redis + backend avec `migrate` au démarrage) | Section 4.10 |
| HTTPS pour toute connexion vers le backend (WebSocket inclus : `wss://`) | Reverse proxy devant Django (HTTP **et** `/ws/`). La liaison Django ↔ PostgreSQL/Redis, interne aux conteneurs, peut rester non chiffrée (le sujet l'autorise) | Section 4.10 |
| README : section « Database Schema » (tables, relations, champs clés et types) | La section 5 de cette doc est un point de départ ; ajouter un schéma visuel | Section 5 |

> Les numéros de modules et leurs points sont ceux de la version précédente de cette doc : à revérifier avec le sujet à jour.
