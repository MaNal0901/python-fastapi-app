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