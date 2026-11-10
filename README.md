# Feature Knowledge Base

Self-contained Generic FeatureCrawler and DialectRanger static viewer. Supply official `urls.json` and a structural `profile.json`; no hosted model key is needed.

## Current datasets

12 registered DBs, 6575 Final records and 1975 separate Review records:

| DB | Final |
|---|---:|
| ClickHouse | 1630 |
| CockroachDB | 782 |
| Derby | 110 |
| DuckDB | 716 |
| MariaDB | 402 |
| MonetDB | 294 |
| MySQL | 483 |
| Oracle | 253 |
| PostgreSQL | 781 |
| SQLite | 225 |
| SQL Server | 396 |
| Trino | 503 |

Counts are published records, not independently certified official recall or precision. The current data includes known content limitations (notably some ClickHouse operator field attribution and page UI text). Updating does not fix these extraction issues.

## Automatic updates (no PR or manual merge)

1. Settings > Actions > General: allow read/write workflow permissions.
2. Settings > Pages: Source = GitHub Actions.
3. Actions > **Update All Feature KBs** > **Run workflow** on main.

The workflow filename remains `.github/workflows/update-sqlite.yml` so existing configuration links keep working, but it now updates **all registered databases**. Each Sunday at 02:17 Asia/Shanghai it reads `data/FeatureKnowledgeBase/catalog.json`, checks every DB has `script/FeatureKnowledgeBase/inputs/<db>/{urls,profile}.json`, then builds each DB on an isolated runner (max two parallel, 180 minutes per DB). Large runs can take hours and consume GitHub Actions quota. Timed-out or failed DBs keep their previous datasets; successful DBs publish together. Final job regenerates the static web asset once, normally pushes to the default branch, and explicitly dispatches **Deploy Feature KB Website**. No forced push or branch-protection bypass is used. Concurrent main edits may reject a push; inspect the failed run rather than overwrite others' changes.

Check **Per-database update results** in the workflow Summary: the overall workflow finishing does NOT mean every DB updated. Each outcome states UPDATED, FAILED, MISSING_ARTIFACT or INVALID_ARTIFACT. Result artifacts expire after 7 days. Neither a failed DB nor a timed-out DB is silently counted as fresh data.

All builds use `semantic_provider=deterministic`. Worker startup disables hosted model client creation. Before publication: every input URL must fetch successfully, no crawl truncation, known zero external model calls, valid lineage/projection, nonempty Final, unique IDs, and at most 10% count drop against the previous dataset. These are update guards, not official completeness certification. No new extraction rules are introduced by this automation.

## Display and update scope

`data/FeatureKnowledgeBase/catalog.json` is the shared display/update registry. Remove a DB entry to stop updating and displaying it; data files may remain. Run export and commit the generated web files to apply display changes. A manual build/publish explicitly re-registers the requested DB.

```powershell
python -m pip install -r script/FeatureKnowledgeBase/requirements.txt
$env:FEATURE_CRAWLER_HTTP_TRUST_ENV = '0'
python script/FeatureKnowledgeBase/run_feature_crawler.py build --dbms sqlite --semantic-provider deterministic
python script/FeatureKnowledgeBase/run_feature_crawler.py export-web
python script/FeatureKnowledgeBase/run_feature_crawler.py serve
```

Local viewer: http://127.0.0.1:8771/ . Online address is shown by the Pages deployment job. The website consumes `web/js/data.js`, generated from catalog-registered Final/Review files, not raw data JSON directly.

To import an existing successful root-project build, without crawling again:

```powershell
python src/script/FeatureKnowledgeBase/run_feature_crawler.py publish --dbms oracle --public-root output/oracle
```

Run this import example from the parent FeatureCrawler directory. Other commands above run from this repository root.

## Layout

- `script/FeatureKnowledgeBase/agents/`: copied Generic Pipeline.
- `script/FeatureKnowledgeBase/crawler_cli.py`: low-level Unified engine.
- `script/FeatureKnowledgeBase/run_feature_crawler.py`: build/publish/export/serve integration.
- `ci_update.py`: one-DB validation and isolated candidate staging.
- `ci_update_all.py`: catalog matrix plan, per-DB worker, final collection.
- `data/FeatureKnowledgeBase/<db>/`: final.json, review.json, portable summary.json.
- `web/`: shared static viewer; descriptions, formal syntax, variants, examples, provenance.

Commit repository contents at root, not inside another src folder. `.gitignore` excludes `.runtime`, .env, caches and virtual environments. Do not upload API secrets or runtime HTML corpora. No personal access token is required by the workflows.

## Tests

The workflow runs 56 selected offline tests covering adapter, isolated model lock, update guards, per-DB failure isolation, count conservation and cached Unified smoke runs. Tests prohibit real network. Historical tests outside this selected suite are retained but are not claimed fully passing. GitHub live crawling must be assessed from actual job results.

GitHub documentation: https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow
