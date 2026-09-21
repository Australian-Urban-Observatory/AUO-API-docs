# Docs live in a separate public repo from the backend

The AUO Data API's user manual needs to be publicly readable by external partner developers.
The backend lives in `AUO-API`, which is private, and the `Australian-Urban-Observatory` org
is on GitHub's Free plan — GitHub Pages cannot publish from a private repo on that plan
(public or private Pages output both require GitHub Pro/Team/Enterprise for a private
source repo).

We created a new **public** repo, `AUO-API-docs`, to hold the MkDocs source and GitHub Pages
deployment, instead of the usual practice of co-locating docs with the code they describe.

**Consequence:** docs and backend now live in separate repos with separate PR histories, so
they can drift out of sync silently. `AUO-API`'s PR template carries a reminder to update
this repo when handler behaviour changes, but there is no automated check enforcing it.

**Alternatives considered:** upgrading the org to GitHub Team (keeps everything in one repo,
costs money); deploying via Netlify/Vercel from the private repo (keeps one repo, adds a
second hosting platform to operate). Both were rejected in favour of the simpler, free,
all-GitHub option, accepting the sync-drift risk as a manual-process cost.
