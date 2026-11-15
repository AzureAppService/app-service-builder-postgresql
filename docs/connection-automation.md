# Configure ShopStream with CLI or ARM

Use a dedicated DEV/TEST group, explicit subscription and approved regional capacity/prices.
Run from your fork's root; do not apply creation templates over a configured app.
Follow README sections 1–2 to deploy `infra/main.bicep`, permit only the approved network path and run administrator `db:bootstrap`.
That template creates the Entra-only server/database/admin; it does not create the runtime database principal.

## Provision the source-less app

The app template uses minimum 0 / maximum 2 HTTP replicas and preserves idle timeout.
Preview requires a minimum of zero; do not disable idle timeout.

Set `SUBSCRIPTION`, `RG` and a unique new `APP`.

```bash
az deployment group create --subscription "$SUBSCRIPTION" -g "$RG" \
  --name shopstream-app --template-file infra/app.bicep --parameters appName="$APP"
az rest --method get --url "https://management.azure.com/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Web/builderApps/$APP?api-version=2026-09-01-preview" \
  --query '{identity:identity,identityUpdate:properties.identityUpdate}'
```

Wait for committed identity and no pending identity operation (`builder app status`).
For ARM-only tooling, GET the app ARM URL below and inspect `identity` and `properties.identityUpdate`.
The administrator must perform README section 3's `db:grant` using this principal ID, never make it a database administrator.
Visitor access is separate; keep writes disabled until authorized.
Copy root `connection.json` to private, ignored `.local/connection.json` and fill the server resource ID/host.
Prepare the workspace with `umask 077; mkdir -p .local; cp connection.json .local/connection.json`.
Keep `PGDATABASE=shopstream`, `PGUSER=shopstream_runtime`, port 5432 and the SAMI selection.
Do not add `PGSSLMODE`, `PGPASSWORD`, a connection URL or password declaration: the sample owns verified TLS and Entra tokens.

## Add with native Builder CLI

Requires a release with `builder app connections --help`; older releases use ARM below.

```bash
builder login
builder app connections list --name "$APP" -s "$SUBSCRIPTION" -g "$RG" --json
builder app connections add --name "$APP" -s "$SUBSCRIPTION" -g "$RG" --connection-json '@.local/connection.json'
builder app connections status --name "$APP" -s "$SUBSCRIPTION" -g "$RG" --connection-name database
```

Do not use `--grant-access` for PostgreSQL: ARM roles cannot grant database privileges.
`update --connection-json` accepts one complete replacement; `remove --connection-name database` removes pending configuration only.
The platform Password alternative is a different client contract, not this sample's default or fallback.

## Add with ARM instead

Use Bash/`jq`, `umask 077` and ignored `.local/`; app metadata contains identifiers.

```bash
APP_ARM="https://management.azure.com/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Web/builderApps/$APP"
APP_URL="$APP_ARM?api-version=2026-09-01-preview"
az rest --method get --url "$APP_URL" > .local/app.json
ETAG="$(jq -er .etag .local/app.json)"
jq --slurpfile next .local/connection.json '
  {properties:{versionConfiguration:{connections:
    ((.properties.versionConfiguration.connections // [] |
      map(select((.name|ascii_downcase) != ($next[0].name|ascii_downcase)))) + $next)}}}
' .local/app.json > .local/update.json
az rest --method patch --url "$APP_URL" --headers "If-Match=$ETAG" --body @.local/update.json
az rest --method get --url "$APP_URL" --query properties.versionConfiguration.connections
```

The PATCH preserves other connections, manual values, source and scaling.
On 412 refresh/reconcile; do not bypass the guard.
Existing manual routing entries must be removed without an intermediate deployment, not copied from secret-free GET metadata.

## Deploy and verify

Use the README's connected GitHub source, or `git archive --format=zip --output=../shopstream-source.zip HEAD` followed by `builder app deploy --name "$APP" -s "$SUBSCRIPTION" -g "$RG" --source-archive ../shopstream-source.zip --manifest ./builder.yaml --json`.
Wait for terminal success. Verify SQL health/seeded reads, least-privilege role and TLS; enable bounded CRUD only with the README's separate authorization.
A saved connection/diagram node is not SQL authentication or persistence proof.
