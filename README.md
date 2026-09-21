# AUO-API-docs

Source for the AUO Data API user manual, published at GitHub Pages.

This is deliberately a separate repo from [`AUO-API`](https://github.com/Australian-Urban-Observatory/AUO-API)
(the backend itself, which is private) — see [`docs/adr/0001-separate-public-docs-repo.md`](docs/adr/0001-separate-public-docs-repo.md)
for why.

## Local preview

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yml`, which builds the site with MkDocs
and publishes it to GitHub Pages.

## Keeping this in sync with AUO-API

This repo has no automated link to the backend's route/handler code. When an endpoint's
request, response, or error shape changes in `AUO-API`, update the matching page under
`docs/reference/` in the same effort — `AUO-API`'s PR template includes a reminder.
