helm install resourcespace . \
  --set db.rootPassword=secret \
  --set db.password=secret \
  --set admin.password=secret \
  --set s3.accessKey=minioadmin \
  --set s3.secretKey=minioadmin \
  --set app.baseUrl=http://resourcespace-dev




**Key design decisions:**

- `config.php` mounted as ConfigMap subPath into `/config/www/resourcespace/include/config.php` (bypasses setup wizard, same as docker-compose)
- Init job uses `node:20-alpine` and installs `mysql-client` + `npm install` at runtime — no custom image build needed
- Init job runs as a `post-install` Helm hook, so it fires after the app is deployed
- Secrets kept separate; sensitive values come from `values.yaml` (override with `--set` or an external secret manager later)