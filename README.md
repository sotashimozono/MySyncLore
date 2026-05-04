# MySyncLore — `deploy` branch

This branch is the **artifacts branch** for [MySyncLore](https://github.com/sotashimozono/MySyncLore).
Auto-populated by the `sync.yml` workflow on every `main` push:

- `articles/` — Zenn-format articles (synced to https://zenn.dev/sotashimozono via Zenn GitHub deploy)
- `public/`  — Qiita-format articles (pushed to https://qiita.com/sotashimozono via qiita-cli)
- `images/`  — image assets per article
- `INDEX.md` — auto-generated index of all published articles
- `log/`     — publish history ledger + analytics snapshots

**Do not edit this branch manually.** All authoring happens on `main` in `drafts/`.
