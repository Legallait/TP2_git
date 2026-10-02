# TP DevOps Correction Docker

## TP 02 : GitHub Actions

### Partie 1 : Setup GitHub Actions (CI)

#### Objectif

Mettre en place une pipeline de Continuous Integration qui build et teste automatiquement le backend (`simple-api`) à chaque push sur `main`, et à chaque pull request.

#### Structure du repo

```
TP2_git/
├── .github/
│   └── workflows/
│       └── main.yml
├── database/
├── http-server/
├── simple-api/
│   └── pom.xml
└── docker-compose.yaml
```

GitHub ne détecte les workflows que dans `.github/workflows/` à la racine du repo.

#### Workflow `main.yml`

```yaml
name: CI devops 2025

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Build and test with Maven
        run: mvn clean verify --file ./simple-api/pom.xml
```

| Élément | Rôle |
|---|---|
| `on.push.branches` | Déclenche la pipeline sur `main` |
| `on.pull_request` | Déclenche la pipeline sur chaque pull request |
| `runs-on: ubuntu-24.04` | Runner hébergé par GitHub, Docker préinstallé |
| `actions/checkout@v4` | Clone le repo dans le runner |
| `actions/setup-java@v4` | Installe le JDK 21 Temurin, avec cache des dépendances Maven |
| `mvn clean verify` | Supprime les anciens builds, compile, lance les unit tests et integration tests |
| `--file ./simple-api/pom.xml` | Indique le `pom.xml`, la commande étant lancée depuis la racine |

#### Problèmes rencontrés

**Testcontainers incompatible avec Docker 29.** En local, les 13 integration tests échouaient avec `Could not find a valid Docker environment`. La version 1.20.4 de Testcontainers utilise une version de l'API Docker refusée par Docker Engine 29. Correction dans `simple-api/pom.xml` :

```xml
<testcontainers.version>1.21.4</testcontainers.version>
```

**Workflow non détecté.** Le fichier avait été créé dans `workflows/main.yml` au lieu de `.github/workflows/main.yml`. Il a été déplacé au bon endroit.

**Warnings de dépréciation.** GitHub signale que `actions/checkout@v4` et `actions/setup-java@v4` reposent sur Node.js 20, déprécié. Le passage en `@v5` les supprime. Les `v4` sont conservées ici car imposées par le sujet.

#### Résultat

- En local : `mvn clean verify` donne `BUILD SUCCESS`, 13 tests passés.
- Sur GitHub Actions : job `test-backend` au vert en 1 min 11 s.

---

### Partie 2 : First steps into the CD World

#### Objectif

Builder les images Docker de l'application dans la pipeline, et les publier sur Docker Hub à chaque commit sur `main`, uniquement si les tests passent.

#### Secrets GitHub

Les identifiants Docker Hub ne sont jamais écrits dans le repo. Ils sont stockés dans Settings > Secrets and variables > Actions :

| Secret | Contenu |
|---|---|
| `DOCKERHUB_USERNAME` | Nom d'utilisateur Docker Hub |
| `DOCKERHUB_TOKEN` | Personal access token Docker Hub (permission Read & Write) |

Un personal access token est utilisé à la place du mot de passe : il peut être révoqué à tout moment sans toucher au compte.

#### Job ajouté au `main.yml`

```yaml
  build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to DockerHub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login --username ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin

      - name: Build image and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./simple-api
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v6
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v6
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

| Élément | Rôle |
|---|---|
| `needs: test-backend` | Le job attend la réussite des tests |
| `docker login --password-stdin` | Connexion à Docker Hub, le token passe par stdin et n'apparaît pas dans la commande |
| `docker/build-push-action@v6` | Build l'image à partir du `Dockerfile` du dossier `context` |
| `tags` | Nom de l'image sur Docker Hub, préfixé par le compte (tout en minuscules) |
| `push: ${{ github.ref == 'refs/heads/main' }}` | Build sur toutes les branches, push uniquement sur `main` |


#### Résultat

- Pipeline en Success en 2 min 32 s : `test-backend` puis `build-and-push-docker-image`.
- Trois images publiées sur Docker Hub avec le tag `latest` :
    - `nicolases/tp-devops-simple-api`
    - `nicolases/tp-devops-database`
    - `nicolases/tp-devops-httpd`