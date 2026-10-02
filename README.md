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

---

### Partie 3 : Setup Quality Gate

#### Objectif

Analyser automatiquement la qualité du code à chaque push avec SonarQube Cloud (anciennement SonarCloud) : bugs, vulnérabilités, code smells, duplications et couverture de tests.

#### Configuration SonarQube Cloud

1. Connexion avec GitHub et import du repo `TP2_git`. L'organisation est créée à l'import.
2. Projet passé en **Public** (Administration > Permissions) : le plan gratuit n'analyse que les projets publics, le repo GitHub a donc aussi été rendu public.
3. **Automatic Analysis désactivée** (Administration > Analysis method) : pour un projet Java, l'analyse doit passer par la CI, et SonarQube Cloud refuse deux méthodes d'analyse en même temps.
4. Génération d'un token (My Account > Access Tokens), ajouté dans GitHub comme secret `SONAR_TOKEN`.

| Paramètre | Valeur |
|---|---|
| Organization key | `legallait` |
| Project key | `Legallait_TP2_git` |
| Quality Gate | Sonar way (par défaut) |

#### Modifications du job `test-backend`

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # ...

      - name: SonarCloud analysis
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Legallait_TP2_git -Dsonar.organization=legallait -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }} --file ./simple-api/pom.xml
```

| Élément | Rôle |
|---|---|
| `fetch-depth: 0` | Récupère tout l'historique git, nécessaire à Sonar pour identifier le nouveau code |
| `-B` | Batch mode, sortie non interactive adaptée à la CI |
| `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` | Lance l'analyse Sonar via le plugin Maven |
| `-Dsonar.projectKey` / `-Dsonar.organization` | Identifient le projet sur SonarQube Cloud |
| `-Dsonar.host.url` | Serveur d'analyse |
| `-Dsonar.token` | Authentification via le secret GitHub |

La couverture de tests est remontée automatiquement grâce au plugin JaCoCo déjà configuré dans le `pom.xml`.

#### Problèmes rencontrés

**Préfixe `sonar` introuvable.** La commande du sujet (`mvn ... sonar:sonar`) échouait avec `No plugin found for prefix 'sonar'` : le plugin Sonar n'est ni déclaré dans le `pom.xml`, ni dans les plugin groups de Maven. Correction : appel du plugin par son nom complet `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`.

**`sonar.login` déprécié.** Remplacé par `sonar.token`, le paramètre attendu par les versions récentes du scanner.

**Projet privé.** Le plan gratuit refuse l'analyse des projets privés : le repo GitHub et le projet SonarQube Cloud ont été passés en public. Les secrets GitHub restent chiffrés et invisibles.

#### Résultat

- Pipeline au vert en 2 min 37 s : tests, analyse Sonar, puis build et push des images.
- Rapport disponible sur SonarQube Cloud avec le statut de la Quality Gate, les notes Reliability, Security et Maintainability, et la couverture de tests.
