# Lightning

## Repository layout

This workspace contains **three independent git repositories**:

| Path | Repo |
|------|------|
| `./` (root) | docs, notes, plans |
| `./client` | frontend (own `.git`) |
| `./server` | backend (own `.git`) |

`client/` and `server/` are listed in the root `.gitignore` because they are nested git repos, not subdirectories of the root repo. To commit work under those directories, `cd` into them and run git there. Running `git status` at the root will show nothing for edits under `client/` or `server/` — that is expected.
