# FastAPI Starter

A minimal, production‑leaning FastAPI template with testing (coverage), linting/formatting, Docker, and CI. Ships with a root endpoint and a health check.

## Features
- FastAPI + Uvicorn
- Pydantic Settings for configuration management
- Custom logging with TRACE level support
- Testing with pytest + pytest-cov (HTML/XML coverage, 85% threshold)
- Ruff for linting and formatting; optional mypy typing
- Dockerfile and GitHub Actions CI (Python 3.13)

## Endpoints
- `GET /` → `{ "message": "the server is now active" }`
- `GET /health` → `{ "ok": true }`

## Quick Start
```bash
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
cp .env.example .env  # Optional: customize your environment variables
uvicorn app.main:app --reload --port 8000
```

Open [http://localhost:8000](http://localhost:8000) and [http://localhost:8000/health](http://localhost:8000/health)

## Testing with Coverage

```bash
pytest
# Outputs: htmlcov/ (HTML), coverage.xml (XML), threshold enforced via pyproject.toml
```

## Linting and Formatting

```bash
ruff check .
ruff format .
# Optional typing
mypy app
```

## Docker

```bash
docker build -t fastapi-starter .
docker run -p 8000:8000 fastapi-starter
```

## CI

GitHub Actions workflow runs linting and tests with coverage on Python 3.13. See `.github/workflows/ci.yml`.
Test TP5 : modification du README.

## License

MIT. See [LICENSE](./LICENSE).

---

# TP6 – Variables d'environnement, Inputs et Outputs (GitHub Actions)

**Branche :** `feature/tp-env`
**Workflows :** [`tp-env.yml`](./.github/workflows/tp-env.yml) et [`greeting.yml`](./.github/workflows/greeting.yml)

## Contenu du TP

| Partie | Ce qui a été réalisé |
|---|---|
| 0 | Création de la branche `feature/tp-env` et du squelette `tp-env.yml` (déclencheur `workflow_dispatch`) |
| 1 | Variables `APP_NAME` et `BUILD_DATE` définies avec `GITHUB_ENV`, puis affichées dans le step suivant |
| 2 | Inputs `environment` (choice), `version` (avec valeur par défaut) et `debug` (boolean), avec un step conditionnel `if: ${{ inputs.debug }}` |
| 3 | Output `full_version` produit par le job `build` (step `Compute full version`, `id: version`, `GITHUB_OUTPUT`) et réutilisé par le job `deploy` (`needs: build`) |
| 4 | Workflow réutilisable `greeting.yml` (`workflow_call`, input `recipient`), appelé depuis `tp-env.yml` par le job `call-greeting` |

## Réponses aux questions

**Q1. Pourquoi la variable n'est-elle pas disponible dans la step qui l'a définie, mais seulement dans les suivantes ?**
`GITHUB_ENV` est un fichier géré par le runner. Quand une step écrit `NOM=valeur` dedans, le runner ne lit ce fichier qu'à la fin de la step. L'environnement de la step en cours est déjà figé au démarrage, donc la variable n'existe pas encore pour elle. Elle est ajoutée à l'environnement des steps suivantes du même job.

**Q2. Que constate-t-on dans le job `verify-env` ? Quelle est la portée de `GITHUB_ENV` ?**
`$APP_NAME` s'affiche vide (`APP_NAME=[]`). Chaque job s'exécute sur son propre runner, c'est-à-dire sur une machine virtuelle distincte, et ne voit donc pas les variables d'un autre job. La portée de `GITHUB_ENV` se limite aux steps suivantes **du même job**.

**Q3. Que se passe-t-il avec `>` au lieu de `>>` dans `GITHUB_OUTPUT` quand deux outputs sont définis dans la même step ?**
Avec `>`, le fichier est écrasé à chaque écriture : seule la dernière ligne est conservée, et le premier output est perdu (il est vide dans les jobs suivants). Il faut utiliser `>>` pour ajouter une ligne à la fin du fichier sans effacer les précédentes.

**Q4. Pourquoi `GITHUB_ENV` ne suffit-il pas ici, alors que `GITHUB_OUTPUT` fonctionne entre jobs ?**
`GITHUB_ENV` ne vit que dans un seul job (un seul runner), et les jobs sont isolés les uns des autres. `GITHUB_OUTPUT` fonctionne autrement : la valeur est déclarée dans les `outputs:` du job, GitHub la conserve, puis la met à disposition des jobs qui dépendent de lui via `needs.<job>.outputs.<nom>`.

**Q5. Que se passe-t-il si on retire `needs: build` du job `deploy` ?**
Le job `deploy` ne dépend plus de `build` : les deux jobs démarrent en parallèle, sans garantie d'ordre. Comme `build` n'est plus déclaré dans `needs`, `needs.build.outputs.full_version` est vide dans `deploy`, donc la version n'est pas affichée.

## Exécution

Le workflow `Env, Inputs and Outputs Lab` se lance manuellement depuis l'onglet **Actions** (`workflow_dispatch`), en choisissant la branche `feature/tp-env` et les paramètres `environment`, `version` et `debug`. Les jobs `build`, `deploy`, `verify-env` et `call-greeting / greet` s'exécutent avec succès.

# TP7 – Actions tierces et actions personnalisées

**Q1. Pourquoi `actions/checkout` n'a-t-il pas besoin de `with` alors que `setup-node` en a besoin ?**
Tous les paramètres de `checkout` ont des valeurs par défaut adaptées (le dépôt courant, la branche ou le commit du workflow, le jeton par défaut), donc il fonctionne sans configuration. `setup-node` doit savoir quelle version de Node.js installer : ce choix dépend de mon projet, il faut donc le préciser avec `with: node-version: 20`.

**Q2. Pourquoi le job `verify` ne retrouve-t-il pas `report.txt` sans l'artefact, alors que les deux jobs partagent le même dépôt ?**
Chaque job s'exécute sur un runner distinct et neuf. `report.txt` est un fichier généré pendant le job `build` : il n'est pas dans le dépôt et n'existe que dans l'espace de travail de ce runner. Le dépôt partagé ne contient que les fichiers commités. Pour transmettre un fichier d'un job à l'autre, il faut passer par un artefact (upload puis download).

**Q6. Pourquoi `actions/checkout` doit-il s'exécuter avant l'appel d'une action locale ?**
Une action locale (`./.github/actions/build-info`) est un dossier du dépôt. Sur un runner neuf, l'espace de travail est vide tant que `checkout` n'a pas récupéré le code. Sans lui, le runner ne trouve pas le fichier `action.yml` et le workflow échoue.

**Q7. Pourquoi `shell: bash` est-il obligatoire sur les steps `run` d'une action composite ?**
Dans une action composite, les steps `run` n'héritent pas d'un shell par défaut : le shell n'est pas repris du workflow appelant, et l'action peut être utilisée sur différents systèmes. Il faut donc indiquer explicitement le shell sur chaque step `run`, sinon l'action est refusée.

## Partie 4 – Comparaison action Docker et action composite
Dans mon run, l'action composite (`build-info`) a pris environ X s, alors que l'action Docker (`docker-hello`) a pris environ Y s. L'action Docker est plus lente car GitHub doit construire l'image à partir du `Dockerfile` à chaque exécution, puis démarrer un conteneur. L'action composite lance simplement des commandes sur le runner. En contrepartie, l'action Docker offre un environnement isolé et entièrement maîtrisé.


# TP8 – Stratégies GitHub Actions

**Q1. Combien de jobs sont générés ? Donnez la liste des combinaisons.**
6 jobs (2 systèmes × 3 versions de Python) : ubuntu-latest avec 3.11, 3.12 et 3.13, et windows-latest avec 3.11, 3.12 et 3.13.

**Q2. Combien de jobs au total ?**
6 jobs. La matrice de base compte 6 combinaisons ; `exclude` en retire une (windows-latest + 3.11), ce qui en laisse 5 ; `include` en ajoute une (ubuntu-latest + 3.14, expérimentale), ce qui donne 6. L'exclusion est appliquée avant l'inclusion.

**Q3. Que se passerait-il si on tentait d'exclure la combinaison 3.14 ? Pourquoi ?**
Rien : le job 3.14 s'exécuterait quand même. `exclude` s'applique à la matrice de base, avant `include`. À ce moment-là, la combinaison 3.14 n'existe pas encore (elle est ajoutée ensuite par `include`), donc il n'y a rien à exclure.

**Q4. Quelle est la différence observée entre `fail-fast: true` et `fail-fast: false` ?**
Avec `fail-fast: true`, quand le job windows-latest 3.13 a échoué (test_break), GitHub a annulé le job windows-latest 3.12 qui tournait encore : 4 jobs réussis, 1 échoué, 1 annulé. Avec `fail-fast: false`, tous les jobs sont allés jusqu'au bout : 4 jobs réussis (Ubuntu), 2 échoués (Windows), 0 annulé. Je choisis `true` pour les pull requests courantes : un échec suffit à bloquer, donc on économise du temps et des ressources de calcul. Je choisis `false` quand je veux connaître toutes les combinaisons qui échouent, par exemple pour tester la compatibilité sur plusieurs systèmes ou versions.

**Q5. Deux raisons d'utiliser `max-parallel` en entreprise**
1. Maîtriser les coûts et les quotas de calcul : limiter le nombre de runners utilisés en même temps pour ne pas saturer les ressources partagées avec les autres équipes.
2. Protéger une ressource partagée (base de test, serveur de staging, API externe limitée) qu'un trop grand nombre de jobs simultanés pourrait surcharger.
