## TP 02 : GitHub Actions

#### Structure du repo final après tous les bonus

```
TP2_git/
├── .github/
│   └── workflows/
│       └── main.yml
│       └── build-and-push.yml
│       └── test-backend.yml
├── database/
├── http-server/
├── simple-api/
│   └── pom.xml
└── docker-compose.yaml
```

GitHub ne détecte les workflows que dans `.github/workflows/` à la racine du repo.

### Partie 1 : Setup GitHub Actions (CI)

#### Objectif

Mettre en place une pipeline de Continuous Integration qui build et teste automatiquement le backend (`simple-api`) à
chaque push sur `main`, et à chaque pull request.

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

| Élément                       | Rôle                                                                            |
|-------------------------------|---------------------------------------------------------------------------------|
| `on.push.branches`            | Déclenche la pipeline sur `main`                                                |
| `on.pull_request`             | Déclenche la pipeline sur chaque pull request                                   |
| `runs-on: ubuntu-24.04`       | Runner hébergé par GitHub, Docker préinstallé                                   |
| `actions/checkout@v4`         | Clone le repo dans le runner                                                    |
| `actions/setup-java@v4`       | Installe le JDK 21 Temurin, avec cache des dépendances Maven                    |
| `mvn clean verify`            | Supprime les anciens builds, compile, lance les unit tests et integration tests |
| `--file ./simple-api/pom.xml` | Indique le `pom.xml`, la commande étant lancée depuis la racine                 |

#### Résultat

- En local : `mvn clean verify` donne `BUILD SUCCESS`, 13 tests passés.
- Sur GitHub Actions : job `test-backend` au vert en 1 min 11 s.

---

### Partie 2 : First steps into the CD World

#### Objectif

Créer les images Docker de l'application dans la pipeline, et les publier sur Docker Hub à chaque commit sur `main`,
uniquement si les tests passent.

#### Secrets GitHub

Les identifiants Docker Hub ne sont jamais écrits dans le repo. Ils sont stockés dans Settings > Secrets and variables >
Actions :

| Secret               | Contenu                                                    |
|----------------------|------------------------------------------------------------|
| `DOCKERHUB_USERNAME` | Nom d'utilisateur Docker Hub                               |
| `DOCKERHUB_TOKEN`    | Personal access token Docker Hub (permission Read & Write) |

Un personal access token est utilisé à la place du mot de passe : il peut être révoqué à tout moment sans toucher au
compte.

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

| Élément                                        | Rôle                                                                                |
|------------------------------------------------|-------------------------------------------------------------------------------------|
| `needs: test-backend`                          | Le job attend la réussite des tests                                                 |
| `docker login --password-stdin`                | Connexion à Docker Hub, le token passe par stdin et n'apparaît pas dans la commande |
| `docker/build-push-action@v6`                  | Build l'image à partir du `Dockerfile` du dossier `context`                         |
| `tags`                                         | Nom de l'image sur Docker Hub, préfixé par le compte (tout en minuscules)           |
| `push: ${{ github.ref == 'refs/heads/main' }}` | Build sur toutes les branches, push uniquement sur `main`                           |

#### Résultat

- Pipeline en Success en 2 min 32 s : `test-backend` puis `build-and-push-docker-image`.
- Trois images publiées sur Docker Hub avec le tag `latest` :
    - `nicolases/tp-devops-simple-api`
    - `nicolases/tp-devops-database`
    - `nicolases/tp-devops-httpd`

---

### Partie 3 : Setup Quality Gate

#### Objectif

Analyser automatiquement la qualité du code à chaque push avec SonarQube Cloud (anciennement SonarCloud) : bugs,
vulnérabilités, code smells, duplications et couverture de tests.

#### Modifications du job `test-backend`

```yaml
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # ...

      - name: SonarCloud analysis
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Legallait_TP2_git -Dsonar.organization=legallait -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }} --file ./simple-api/pom.xml
```

| Élément                                                  | Rôle                                                                               |
|----------------------------------------------------------|------------------------------------------------------------------------------------|
| `fetch-depth: 0`                                         | Récupère tout l'historique git, nécessaire à Sonar pour identifier le nouveau code |
| `-B`                                                     | Batch mode, sortie non interactive adaptée à la CI                                 |
| `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` | Lance l'analyse Sonar via le plugin Maven                                          |
| `-Dsonar.projectKey` / `-Dsonar.organization`            | Identifient le projet sur SonarQube Cloud                                          |
| `-Dsonar.host.url`                                       | Serveur d'analyse                                                                  |
| `-Dsonar.token`                                          | Authentification via le secret GitHub                                              |

#### Résultat

- Pipeline au vert en 2 min 37 s : tests, analyse Sonar, puis build et push des images.
- Rapport disponible sur SonarQube Cloud avec le statut de la Quality Gate, les notes Reliability, Security et
  Maintainability, et la couverture de tests.

---

### Partie 4 : Going further, Split pipelines

#### Objectif

Séparer la pipeline unique en deux workflows :

- `test-backend` lancé sur `develop` et `main` ;
- `build-and-push-docker-image` lancé sur `main` uniquement, et seulement si `test-backend` a réussi.

#### Nouvelle structure

```
.github/
└── workflows/
    ├── test-backend.yml
    └── build-and-push.yml
```

Le fichier `main.yml` a été supprimé, et une branche `develop` a été créée.

#### Workflow `test-backend.yml`

```yaml
name: Test backend

on:
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Build and test with Maven
        run: mvn clean verify --file ./simple-api/pom.xml

      - name: SonarCloud analysis
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Legallait_TP2_git -Dsonar.organization=legallait -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }} --file ./simple-api/pom.xml
```

Ce workflow ne fait que tester et analyser le code. Il ne publie aucune image.

#### Workflow `build-and-push.yml`

```yaml
name: Build and push docker images

on:
  workflow_run:
    workflows: [ "Test backend" ]
    types:
      - completed
    branches:
      - main

jobs:
  build-and-push-docker-image:
    if: ${{ github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push' }}
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}

      - name: Login to DockerHub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login --username ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin

      - name: Build image and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./simple-api
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          push: true

      - name: Build image and push database
        uses: docker/build-push-action@v6
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          push: true

      - name: Build image and push httpd
        uses: docker/build-push-action@v6
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-httpd:latest
          push: true
```

| Élément                     | Rôle                                                                         |
|-----------------------------|------------------------------------------------------------------------------|
| `on.workflow_run.workflows` | Surveille le workflow `Test backend`, le nom doit correspondre exactement    |
| `types: completed`          | Déclenche à la fin du workflow surveillé, qu'il ait réussi ou échoué         |
| `branches: main`            | Ne réagit qu'aux runs de `Test backend` sur `main`                           |
| `conclusion == 'success'`   | Ne build que si les tests ont réussi                                         |
| `event == 'push'`           | Ne publie jamais d'image à partir d'une pull request                         |
| `ref: ...head_sha`          | Checkout du commit exact qui a été testé, et non du dernier commit de `main` |
| `push: true`                | Le filtre `branches: main` garantit déjà que l'on est sur `main`             |

#### Comportement obtenu

|                              | `develop` | `main`                       |
|------------------------------|-----------|------------------------------|
| Test backend                 | Oui       | Oui                          |
| Build and push docker images | Non       | Oui, si les tests ont réussi |

- `workflow_run` ne fonctionne que si le fichier du workflow déclenché est présent sur la branche par défaut (`main`).

#### Résultat

- `Test backend` au vert sur `main` et sur `develop`.
- `Build and push docker images` déclenché automatiquement après le run sur `main`, au vert en 1 min 24 s.
- Aucun build d'image déclenché par le run sur `develop`.

---

### Bonus : Version tags sur les images

#### Objectif

Ne plus publier uniquement `latest`, qui est écrasé à chaque push : chaque image reçoit aussi un tag de version
immuable, ce qui permet de savoir quel commit tourne et de revenir à une version précédente. Le push des images reste
limité aux push et merge sur `main`.

| Option              | Retenue | Raison                                                                     |
|---------------------|---------|----------------------------------------------------------------------------|
| Tag git `vX.Y.Z`    | Non     | Déclenche la pipeline sur un tag et non sur `main`, contraire à l'objectif |
| `github.run_number` | Non     | Lié au numéro de run, pas au code : un re-run change la version            |
| SHA du commit       | Oui     | Unique, immuable, et relie directement l'image au commit sur GitHub        |

#### Modifications de `build-and-push.yml`

Ajout d'une étape qui calcule le tag, avant le login :

```yaml
      - name: Compute image version
        id: version
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          echo "tag=sha-${SHA::7}" >> "$GITHUB_OUTPUT"
```

Chaque `build-push-action` publie désormais deux tags :

```yaml
      - name: Build image and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./simple-api
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:${{ steps.version.outputs.tag }}
          push: true
```

Même modification pour `database` et `httpd`. Le déclenchement (`workflow_run` sur `main`, `conclusion == 'success'`,
`event == 'push'`) et `test-backend.yml` sont inchangés.

| Élément                 | Rôle                                                                     |
|-------------------------|--------------------------------------------------------------------------|
| `id: version`           | Permet de référencer la sortie de l'étape avec `steps.version.outputs`   |
| `workflow_run.head_sha` | SHA du commit testé par `Test backend`, et non celui du workflow courant |
| `${SHA::7}`             | Garde les 7 premiers caractères, comme l'affichage court de git          |
| `$GITHUB_OUTPUT`        | Expose la valeur `tag` aux étapes suivantes                              |
| `tags: \|`              | Liste multi-ligne : une même image poussée sous plusieurs tags           |

#### Comportement obtenu

| Événement                          | Pipeline de push | Tags publiés              |
|------------------------------------|------------------|---------------------------|
| Push sur `develop`                 | Non              | Aucun                     |
| Pull request                       | Non              | Aucun                     |
| Push ou merge sur `main`, tests OK | Oui              | `latest` et `sha-xxxxxxx` |
| Push ou merge sur `main`, tests KO | Non              | Aucun                     |

#### Résultat

- Trois images publiées sur Docker Hub avec deux tags à chaque push sur `main` :
    - `nicolases/tp-devops-simple-api:latest` et `:sha-xxxxxxx`
    - `nicolases/tp-devops-database:latest` et `:sha-xxxxxxx`
    - `nicolases/tp-devops-httpd:latest` et `:sha-xxxxxxx`

---

### Bonus : Rollback si les tests échouent

#### Objectif

Si un push ou un merge sur `main` casse les tests, on remet automatiquement `main` dans le dernier état qui passait, au
lieu de laisser la branche principale cassée.

Les images ne sont déjà pas publiées quand les tests échouent : `latest` pointe donc toujours sur la dernière version
validée. Le rollback porte sur le code de `main`.

#### Job ajouté à `build-and-push.yml`

```yaml
  rollback:
    if: ${{ github.event.workflow_run.conclusion == 'failure' && github.event.workflow_run.event == 'push' }}
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - name: Checkout main
        uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      - name: Revert failing commit
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          if [ "$(git rev-list --parents -n 1 "$SHA" | wc -w)" -gt 2 ]; then
            git revert --no-edit -m 1 "$SHA"
          else
            git revert --no-edit "$SHA"
          fi
          git push origin main
```

| Élément                        | Rôle                                                                                     |
|--------------------------------|------------------------------------------------------------------------------------------|
| `conclusion == 'failure'`      | Le job ne tourne que si `Test backend` a échoué, à l'inverse du job de build             |
| `event == 'push'`              | Pas de rollback pour une pull request, qui n'a rien modifié sur `main`                   |
| `permissions: contents: write` | Autorise le `GITHUB_TOKEN` à pousser sur `main`                                          |
| `ref: main` / `fetch-depth: 0` | Récupère `main` avec tout l'historique, nécessaire à `git revert`                        |
| `git config user.*`            | Identité du bot qui signe le commit de revert                                            |
| `git rev-list --parents`       | Compte les parents du commit : plus de deux mots signifie un merge commit                |
| `git revert -m 1`              | Pour un merge, annule les changements par rapport au premier parent, c'est-à-dire `main` |
| `git revert --no-edit`         | Crée un nouveau commit inverse, sans réécrire l'historique                               |
| `git push origin main`         | Publie le revert sur `main`                                                              |

`git revert` est préféré à `git reset` : l'historique reste intact, le commit fautif reste visible et peut être corrigé
puis réappliqué.

#### Comportement obtenu

| Résultat de `Test backend` sur `main` | Job exécuté                   |
|---------------------------------------|-------------------------------|
| Success                               | `build-and-push-docker-image` |
| Failure                               | `rollback`                    |

- Un push fait avec le `GITHUB_TOKEN` ne déclenche pas de nouveau workflow : le revert ne relance pas `Test backend`, ce
  qui évite toute boucle.
- Si plusieurs commits sont poussés en une fois, seul le dernier est annulé.
- Si `main` est protégée par une branch protection qui interdit le push direct, le rollback échoue : il faut autoriser
  GitHub Actions dans la règle.

#### Résultat

- Un commit qui casse les tests sur `main` est annulé automatiquement par un commit `Revert "..."` signé
  `github-actions[bot]`.
- Aucune image n'est publiée pour ce commit, Docker Hub garde la dernière version validée.

---

### Bonus : Simplification des workflows

#### Objectif

Réduire la taille et la répétition des deux workflows, sans changer leur comportement : mêmes déclencheurs, mêmes
conditions, mêmes images publiées, même rollback.

#### Workflow `test-backend.yml`

```yaml
name: Test backend

on:
  # tests run on main and develop
  push:
    branches: [ main, develop ]
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-24.04
    steps:
      # checkout with full git history, required by SonarCloud
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # setup JDK 21 (Temurin) with Maven dependency cache
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      # build + unit and integration tests (Testcontainers) + quality gate on SonarCloud
      - name: Build, test and SonarCloud
        working-directory: simple-api
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Legallait_TP2_git -Dsonar.organization=legallait -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

#### Workflow `build-and-push.yml`

```yaml
name: Build and push docker images

on:
  # triggered when the "Test backend" workflow ends on main
  workflow_run:
    workflows: [ "Test backend" ]
    types: [ completed ]
    branches: [ main ]

jobs:
  build-and-push:
    # run only if tests passed, and only for a push or merge on main (not a pull request)
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push'
    runs-on: ubuntu-24.04
    # one job per image: backend, database, httpd
    strategy:
      matrix:
        include:
          - { image: simple-api, context: simple-api }
          - { image: database, context: database }
          - { image: httpd, context: http-server }
    steps:
      # checkout the exact commit that was tested
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}

      # login to Docker Hub with GitHub secrets
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # build and push with two tags: latest and the tested commit SHA
      - uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.context }}
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-${{ matrix.image }}:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-${{ matrix.image }}:${{ github.event.workflow_run.head_sha }}

  rollback:
    # run only if tests failed after a push or merge on main
    if: github.event.workflow_run.conclusion == 'failure' && github.event.workflow_run.event == 'push'
    runs-on: ubuntu-24.04
    # allows the GITHUB_TOKEN to push the revert commit on main
    permissions:
      contents: write
    steps:
      # checkout main with full history, required by git revert
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      # revert the failing commit (-m 1 for a merge, plain revert otherwise) and push it on main
      - run: |
          SHA=${{ github.event.workflow_run.head_sha }}
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git revert --no-edit -m 1 $SHA || git revert --no-edit $SHA
          git push
```

#### Changements

| Avant                                             | Après                                        | Gain                                                                                                  |
|---------------------------------------------------|----------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `mvn clean verify` puis `mvn -B verify ...:sonar` | Une seule commande `mvn -B verify ...:sonar` | Le projet n'est compilé et testé qu'une fois                                                          |
| `--file ./simple-api/pom.xml`                     | `working-directory: simple-api`              | Commande plus courte                                                                                  |
| `echo ... \| docker login --password-stdin`       | `docker/login-action@v3`                     | Action officielle, même sécurité du token                                                             |
| 3 steps `build-push-action` copiés-collés         | `strategy.matrix` avec 3 entrées             | Un seul step, les 3 images buildées en parallèle                                                      |
| Step `Compute image version` (`sha-${SHA::7}`)    | `head_sha` utilisé directement dans `tags`   | Un step en moins                                                                                      |
| Test `git rev-list --parents` + `if/else`         | `git revert -m 1 $SHA \|\| git revert $SHA`  | Si le commit n'est pas un merge, `-m 1` échoue sans rien modifier et le revert simple prend le relais |
| Listes YAML sur plusieurs lignes                  | Listes en ligne `[main, develop]`            | Même sens, plus compact                                                                               |

Le `clean` est supprimé car le runner est neuf à chaque exécution : il n'y a aucun ancien build à effacer.

#### Comportement conservé

| Événement                          | Test backend | Build and push                 | Rollback |
|------------------------------------|--------------|--------------------------------|----------|
| Push sur `develop`                 | Oui          | Non                            | Non      |
| Pull request                       | Oui          | Non                            | Non      |
| Push ou merge sur `main`, tests OK | Oui          | Oui, `latest` et SHA du commit | Non      |
| Push ou merge sur `main`, tests KO | Oui          | Non                            | Oui      |

#### Différences

- Le tag de version est le SHA complet du commit au lieu de `sha-xxxxxxx`.
- Les 3 images sont buildées en parallèle. Si une échoue, les autres sont annulées (comportement `fail-fast` par défaut
  de la matrix), ce qui équivaut à l'arrêt au premier échec de la version séquentielle.

#### Résultat

- Workflows plus courts, sans duplication pour les images.
- Images publiées sur Docker Hub avec deux tags à chaque push sur `main` :
    - `nicolases/tp-devops-simple-api:latest` et `:<sha>`
    - `nicolases/tp-devops-database:latest` et `:<sha>`
    - `nicolases/tp-devops-httpd:latest` et `:<sha>`

---

### Bonus : Orchestration avec `main.yml`

#### Objectif

Remplacer le chaînage par `workflow_run` par un workflow principal unique, `main.yml`, qui appelle les deux autres comme
des reusable workflows. L'enchaînement tests, publication et rollback est visible dans un seul graphe GitHub Actions.

#### Nouvelle structure

```
.github/
└── workflows/
    ├── main.yml
    ├── test-backend.yml
    └── build-and-push.yml
```

Seul `main.yml` réagit aux push et pull requests. `test-backend.yml` et `build-and-push.yml` ne se lancent plus seuls.

#### Workflow `main.yml`

```yaml
name: CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:

jobs:
  # tests and quality gate on every push and pull request
  test:
    uses: ./.github/workflows/test-backend.yml
    secrets: inherit

  # publish images only if tests passed, after a push or merge on main
  build-and-push:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: ./.github/workflows/build-and-push.yml
    secrets: inherit

  # revert the commit if tests failed after a push or merge on main
  rollback:
    needs: test
    if: failure() && github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git revert --no-edit -m 1 ${{ github.sha }} || git revert --no-edit ${{ github.sha }}
          git push
```

| Élément                           | Rôle                                                                                                                   |
|-----------------------------------|------------------------------------------------------------------------------------------------------------------------|
| `on.push` / `on.pull_request`     | Mêmes déclencheurs que l'ancien `test-backend.yml`                                                                     |
| `uses: ./.github/workflows/...`   | Appelle un reusable workflow du repo comme un job                                                                      |
| `secrets: inherit`                | Transmet les secrets du repo au workflow appelé, qui n'y a pas accès par défaut                                        |
| `needs: test`                     | Le job attend la fin de `test`                                                                                         |
| `github.event_name == 'push'`     | Pas de publication ni de rollback pour une pull request                                                                |
| `github.ref == 'refs/heads/main'` | Publication et rollback uniquement sur `main`                                                                          |
| `failure()`                       | Sans cette fonction, un job dont la dépendance a échoué est ignoré : elle autorise `rollback` à tourner après un échec |
| `github.sha`                      | Commit qui a déclenché `main.yml`, donc celui qui vient d'être testé                                                   |

#### Workflow `test-backend.yml`

```yaml
name: Test backend

on:
  # called by main.yml
  workflow_call:

jobs:
  test-backend:
    runs-on: ubuntu-24.04
    steps:
      # checkout with full git history, required by SonarCloud
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # setup JDK 21 (Temurin) with Maven dependency cache
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      # build + unit and integration tests (Testcontainers) + quality gate on SonarCloud
      - name: Build, test and SonarCloud
        working-directory: simple-api
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Legallait_TP2_git -Dsonar.organization=legallait -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

#### Workflow `build-and-push.yml`

```yaml
name: Build and push docker images

on:
  # called by main.yml once tests passed on main
  workflow_call:

jobs:
  build-and-push:
    runs-on: ubuntu-24.04
    # one job per image: backend, database, httpd
    strategy:
      matrix:
        include:
          - { image: simple-api, context: simple-api }
          - { image: database, context: database }
          - { image: httpd, context: http-server }
    steps:
      # checkout the commit that triggered main.yml, the one that was just tested
      - uses: actions/checkout@v4

      # login to Docker Hub with GitHub secrets
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # build and push with two tags: latest and the tested commit SHA
      - uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.context }}
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-${{ matrix.image }}:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-${{ matrix.image }}:${{ github.sha }}
```

#### Changements

| Avant                                                      | Après                                                        | Raison                                                                              |
|------------------------------------------------------------|--------------------------------------------------------------|-------------------------------------------------------------------------------------|
| `test-backend.yml` déclenché par `push` et `pull_request`  | `workflow_call`, appelé par `main.yml`                       | Un seul point d'entrée                                                              |
| `build-and-push.yml` déclenché par `workflow_run`          | `workflow_call` + `needs: test` dans `main.yml`              | Dépendance explicite entre les jobs                                                 |
| `if` sur `workflow_run.conclusion` et `workflow_run.event` | `if` sur `github.event_name` et `github.ref` dans `main.yml` | La réussite des tests est garantie par `needs`                                      |
| `ref: ${{ github.event.workflow_run.head_sha }}`           | Checkout par défaut                                          | Le contexte `github` est celui de `main.yml`, `github.sha` est déjà le commit testé |
| Tag `${{ github.event.workflow_run.head_sha }}`            | Tag `${{ github.sha }}`                                      | Même valeur, accès direct                                                           |
| Job `rollback` dans `build-and-push.yml`                   | Job `rollback` dans `main.yml`, avec `failure()`             | `build-and-push.yml` n'est appelé qu'en cas de succès                               |
| Secrets accessibles directement                            | `secrets: inherit`                                           | Un reusable workflow ne reçoit pas les secrets par défaut                           |

La matrice, le login Docker Hub, les deux tags et la commande de revert sont inchangés.

#### Comportement obtenu

| Événement                          | `test` | `build-and-push`               | `rollback` |
|------------------------------------|--------|--------------------------------|------------|
| Push sur `develop`                 | Oui    | Skipped                        | Skipped    |
| Pull request                       | Oui    | Skipped                        | Skipped    |
| Push ou merge sur `main`, tests OK | Oui    | Oui, `latest` et SHA du commit | Skipped    |
| Push ou merge sur `main`, tests KO | Oui    | Skipped                        | Oui        |

- La contrainte de `workflow_run` (fichier du workflow déclenché obligatoirement présent sur `main`) disparaît.
- Le commit de revert est poussé avec le `GITHUB_TOKEN` : il ne relance pas `main.yml`, ce qui évite toute boucle.

#### Résultat

- Un seul run `main.yml` par push, avec le graphe `test / test-backend` suivi de `Matrix: build-and-push` ou de
  `rollback`.
- Le préfixe `test /` dans le nom du job confirme l'appel du reusable workflow.

---

### Bonus : Analyse de sécurité OWASP

#### Objectif

Ajouter des contrôles de sécurité à la pipeline, sur deux niveaux :

- le **code** du projet, analysé par SonarQube Cloud avec ses règles de sécurité classées selon l'OWASP Top 10 ;

#### Secret GitHub ajouté

| Secret        | Contenu                                                              |
|---------------|----------------------------------------------------------------------|
| `NVD_API_KEY` | Clé API gratuite du NVD (nvd.nist.gov/developers/request-an-api-key) |

#### Workflow `test-backend.yml`

```yaml
name: Test backend

on:
  # called by main.yml
  workflow_call:

jobs:
  test-backend:
    runs-on: ubuntu-24.04
    steps:
      # checkout with full git history, required by SonarCloud
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # setup JDK 21 (Temurin) with Maven dependency cache
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      # build + unit and integration tests (Testcontainers) + SonarCloud analysis
      - name: Build, test and SonarCloud
        working-directory: simple-api
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=Legallait_TP2_git -Dsonar.organization=legallait -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}

      # restore the NVD database from a previous run
      - uses: actions/cache/restore@v4
        with:
          path: ~/.m2/repository/org/owasp/dependency-check-data
          key: nvd-${{ github.run_id }}
          restore-keys: nvd-

      # scan dependencies against the NVD CVE database, fail on CVSS >= 7
      - name: OWASP Dependency-Check
        id: dependency-check
        working-directory: simple-api
        run: mvn -B org.owasp:dependency-check-maven:check -DnvdApiKey=${{ secrets.NVD_API_KEY }} -DfailBuildOnCVSS=7 -Dformat=HTML

      # save the NVD database even if vulnerabilities were found
      - uses: actions/cache/save@v4
        if: always() && steps.dependency-check.outcome != 'skipped'
        with:
          path: ~/.m2/repository/org/owasp/dependency-check-data
          key: nvd-${{ github.run_id }}

      - uses: actions/upload-artifact@v4
        if: always() && steps.dependency-check.outcome != 'skipped'
        with:
          name: dependency-check-report
          path: simple-api/target/dependency-check-report.html
```

| Élément                                  | Rôle                                                                                                   |
|------------------------------------------|--------------------------------------------------------------------------------------------------------|
| `org.owasp:dependency-check-maven:check` | Plugin appelé par son nom complet, aucune modification du `pom.xml` nécessaire                         |
| `-DnvdApiKey`                            | Authentifie les appels à l'API du NVD                                                                  |
| `-DfailBuildOnCVSS=7`                    | Fait échouer le job si une dépendance a une CVE de score CVSS supérieur ou égal à 7 (High ou Critical) |
| `-Dformat=HTML`                          | Génère un rapport lisible dans `target/dependency-check-report.html`                                   |
| `actions/cache/restore`                  | Récupère la base NVD du run précédent grâce au préfixe `nvd-`                                          |
| `actions/cache/save` + `always()`        | Sauvegarde la base même si des CVE font échouer le job                                                 |
| `key: nvd-${{ github.run_id }}`          | Clé unique à chaque run : le cache est toujours réenregistré avec la base à jour                       |
| `outcome != 'skipped'`                   | Pas de sauvegarde ni d'upload si une étape précédente a déjà échoué                                    |
| `actions/upload-artifact`                | Rend le rapport HTML téléchargeable depuis la page du run                                              |

#### Consulter les résultats

| Où                                            | Ce qu'on y trouve                                                     |
|-----------------------------------------------|-----------------------------------------------------------------------|
| Actions > run > `test / test-backend`         | Statut de chaque step et logs bruts (`Tests run: ...`, CVE détectées) |
| Actions > run > section Artifacts             | Rapport HTML `dependency-check-report`                                |
| SonarQube Cloud > Issues, filtre OWASP Top 10 | Vulnérabilités du code classées par catégorie OWASP                   |
| SonarQube Cloud > Security Hotspots           | Code sensible à revoir manuellement                                   |
| Pull request > checks                         | `SonarCloud Code Analysis` avec le statut de la Quality Gate          |