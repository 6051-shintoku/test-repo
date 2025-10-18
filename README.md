# test-repo

## Import external issues

This repository includes a GitHub Actions workflow to import issues from another repository on-demand.

File: `.github/workflows/import-issues.yml`

How to run:

1. (Optional) If the source repository is private, create a personal access token (PAT) with `repo` read scope and add it to this repository's Settings → Secrets as `SOURCE_TOKEN`.
2. In GitHub, open the Actions tab, choose "Import issues from other repo (manual)", then click "Run workflow".
3. Provide `source_owner` and `source_repo`. Optionally set `include_comments` to `false` to skip comments.

Imported issues will have a marker in their body like `Imported from: owner/repo#123` so duplicates are avoided.

弁当販売 A GitHub Codespaces environment with FastApi and Postgres.
