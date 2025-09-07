



# Deployment 
- Always use Staging SSL for deployment. after all is ready then swithc to production SSL
- Ensure deploying node has minimum 8Gb ram to build and push

# Steps
## 1. Create Docker secret for deployment

kubectl -n $NS create secret docker-registry dockerhub-cred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username='<your_dockerhub_user>' \
  --docker-password='<your_pat_or_password>' \
  --docker-email='<you@example.com>'
### Patch deployment to use it if pods are running
kubectl -n $NS patch deployment gibbon-dev-app --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/imagePullSecrets","value":[{"name":"dockerhub-cred"}]}]'

## 2. Create secret for ChatGPT api ( for custom module)
NS=gibbon-dev-deploy
- change key correctly
kubectl -n "$NS" create secret generic gibbon-dev-openai \
  --from-literal=OPENAI_API_KEY='sk-...your-key...'
kubectl -n "$NS" patch secret gibbon-dev-openai -p '{"immutable":true}'
- key rotation
kubectl -n "$NS" delete secret gibbon-dev-openai
kubectl -n "$NS" create secret generic gibbon-dev-openai \
  --from-literal=OPENAI_API_KEY='sk-...new...' 

- if deployment is running
kubectl -n "$NS" rollout restart deployment/gibbon-dev-app
** Tests - execute in k8s master node: 
#!/usr/bin/env bash
set -eu

NS="gibbon-dev-deploy"
SEC="gibbon-dev-openai"

echo "[1] Secret tail from API (masked):"
kubectl -n "${NS}" get secret "${SEC}" -o jsonpath='{.data.OPENAI_API_KEY}' \
  | base64 -d \
  | awk '{print substr($0,1,6)"…"(length>4?substr($0,length-3):$0)}'; echo

echo "[2] Pods using the secret:"
PODS="$(kubectl -n "${NS}" get pod -l app=gibbon-dev -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}')"
for p in ${PODS}; do
  echo " - ${p}:"

  echo "   Secret file in pod:"
  kubectl -n "${NS}" exec "${p}" -c gibbon -- sh -lc '
    set -eu
    ls -l /run/secrets/openai || true
    if [ -f /run/secrets/openai/OPENAI_API_KEY ]; then
      head -c 6 /run/secrets/openai/OPENAI_API_KEY; echo -n "…"; tail -c 4 /run/secrets/openai/OPENAI_API_KEY; echo
    else
      echo "(missing)"
    fi
  '

  echo "   Apache env (should NOT contain OPENAI_API_KEY):"
  kubectl -n "${NS}" exec "${p}" -c gibbon -- sh -lc '
    set -eu
    PID=$(pgrep -xo apache2 || pgrep -xo httpd || true)
    if [ -n "${PID:-}" ]; then
      tr "\0" "\n" < /proc/$PID/environ | grep "^OPENAI_API_KEY=" || echo "(none)"
    else
      echo "(apache not found)"
    fi
  '

  echo "   PHP getApiKey() last4 (via openai.php):"
  kubectl -n "${NS}" exec "${p}" -c gibbon -- sh -lc \
    'php -r '\''require "/var/www/html/gibbon/openai.php"; $k=getApiKey(false); echo $k?substr($k,-4):"MISSING";'\''; echo'
done


*** important note: if the test still show mismatch new - old key. do the following step to completely refesh the key
NS=gibbon-dev-deploy
- Scale to zero to terminate *all* pods (and any stragglers from old ReplicaSets)
kubectl -n "$NS" scale deploy/gibbon-dev-app --replicas=0
kubectl -n "$NS" wait --for=delete pod -l app=gibbon-dev --timeout=180s

- Scale back up
kubectl -n "$NS" scale deploy/gibbon-dev-app --replicas=1
kubectl -n "$NS" rollout status deploy/gibbon-dev-app --timeout=300s


## 3. Create secret for database  (for server side and php code usage)
- change the detail below as needed (metadatas, string data)
NS=gibbon-dev-deploy
kubectl -n "$NS" apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: gibbon-dev-db-secret
  namespace: gibbon-dev-deploy
type: Opaque
stringData:  
  DB_HOST: gibbon-dev-mysql.gibbon-dev-deploy.svc.cluster.local
  DB_PORT: "3306"
  DB_NAME: gibbon
  DB_USER: gibbonuser
  DB_PASSWORD: gibbonpass123   # db.php also support DB_Pass
  DB_PASS: gibbonpass123
  DB_CHARSET: utf8mb4  
  username: gibbonuser         # used by MYSQL_USER in mysql Deployment
  password: gibbonpass123      # used by MYSQL_PASSWORD & MYSQL_ROOT_PASSWORD
EOF
- in jenkinfile ensure to remove the line below because the deployment will read secret directly from k8s envrionment
kubectl apply -n "${NS}" -f k8s/gibbon-mysql-secret.yaml

- (Optional) check after rollout or deploy
APP_POD=$(kubectl -n gibbon-dev-deploy get pod -l app=gibbon-dev -o jsonpath='{.items[0].metadata.name}')
kubectl -n gibbon-dev-deploy exec "$APP_POD" -c gibbon -- \
  sh -lc 'env | egrep "^DB_(HOST|PORT|NAME|USER|PASS|PASSWORD|CHARSET)="'

## 4. Deploy
- modify all other parameter so that deployment is dedicated to a system (the example is gibbon-dev.vanhoa.edu.vn)
### 4.1. recreate config.php
#!/usr/bin/env bash
set -euo pipefail

 --- Vars (dev) ---
NS="gibbon-dev-deploy"
DEP="gibbon-dev-app"
PVC="gibbon-dev-uploads-pvc"

echo "== 1) Scale app down =="
kubectl -n "$NS" scale deploy/$DEP --replicas=0
kubectl -n "$NS" wait --for=delete pod -l app=gibbon-dev --timeout=180s || true

echo "== 2) Start a helper pod that mounts the same PVC =="
kubectl -n "$NS" delete pod pvc-tool --ignore-not-found
kubectl -n "$NS" run pvc-tool --image=busybox:1.36 --restart=Never \
  --overrides="$(
    cat <<'JSON'
{
  "spec":{
    "volumes":[{"name":"uploads","persistentVolumeClaim":{"claimName":"gibbon-dev-uploads-pvc"}}],
    "containers":[{"name":"sh","image":"busybox:1.36","command":["sh","-c","sleep 3600"],
      "volumeMounts":[{"name":"uploads","mountPath":"/pvc"}]}]
  }
}
JSON
)"
kubectl -n "$NS" wait pod pvc-tool --for=condition=Ready --timeout=60s

echo "== 3) Delete config.php from PVC =="
kubectl -n "$NS" exec pvc-tool -- sh -lc 'rm -f /pvc/config/config.php && ls -l /pvc/config || true'

echo "== 4a) Temporarily remove the config.php subPath volumeMount from the app container =="
- Replace the volumeMounts list with the same mounts but WITHOUT the config.php subPath.
- (Keeps uploads + i18n + openai-secret).
kubectl -n "$NS" patch deploy/$DEP --type=merge -p='
spec:
  template:
    spec:
      containers:
      - name: gibbon
        volumeMounts:
        - name: uploads
          mountPath: /var/www/html/gibbon/uploads
        - name: uploads
          mountPath: /var/www/html/gibbon/i18n
          subPath: i18n
        - name: openai-secret
          mountPath: /run/secrets/openai
          readOnly: true
'

echo "== 4b) Replace init script to SKIP creating config.php (note the # comment, no leading '-') =="
- Only adjust the first initContainer args[0], keeping the rest of the pod spec intact.
PATCH_SCRIPT=$'set -ex\nmkdir -p /pvc/config /pvc/i18n /var/www/html/gibbon/uploads/tmp /var/www/html/gibbon/uploads/cache\n# installer mode: DO NOT create /pvc/config/config.php here\nchown -R 33:33 /var/www/html/gibbon/uploads || true\nchmod -R 0777 /var/www/html/gibbon/uploads || true\n'
kubectl -n "$NS" patch deploy/$DEP --type=json -p="$(
  python3 - <<'PY'
import json, os
script = os.environ["PATCH_SCRIPT"]
print(json.dumps([
  {"op":"replace","path":"/spec/template/spec/initContainers/0/args/0","value":script}
]))
PY
)" || {
  echo "JSON patch failed; falling back to direct inline JSON…"
  kubectl -n "$NS" patch deploy/$DEP --type=json -p="[
    {\"op\":\"replace\",\"path\":\"/spec/template/spec/initContainers/0/args/0\",
     \"value\":\"$PATCH_SCRIPT\"}
  ]"
}

echo "== 5) Scale up one replica and wait =="
kubectl -n "$NS" scale deploy/$DEP --replicas=1
kubectl -n "$NS" rollout status deploy/$DEP --timeout=300s

echo "== 5b) Open the installer in your browser =="
echo "   https://gibbon-dev.vanhoa.edu.vn/installer/"
echo "   Complete all steps (the installer writes /var/www/html/gibbon/config.php INSIDE the pod)."
read -p "Press Enter AFTER you finish the web installer…"

echo "== 6) Persist the generated config.php into the PVC =="
APP_POD="$(kubectl -n "$NS" get pod -l app=gibbon-dev -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" exec "$APP_POD" -c gibbon -- sh -lc 'cat /var/www/html/gibbon/config.php' \
 | kubectl -n "$NS" exec pvc-tool -- sh -lc 'mkdir -p /pvc/config && cat > /pvc/config/config.php && chown 33:33 /pvc/config/config.php && chmod 660 /pvc/config/config.php'
kubectl -n "$NS" exec pvc-tool -- sh -lc 'ls -l /pvc/config/config.php && head -n 3 /pvc/config/config.php || true'

echo "== 7) Restore Deployment to normal (re-add config.php subPath and original init args) =="
- Re-add the config.php subPath mount
kubectl -n "$NS" patch deploy/$DEP --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/2",
   "value":{"name":"uploads","mountPath":"/var/www/html/gibbon/config.php","subPath":"config/config.php","readOnly":true}}
]'

- Restore the original init script that creates config.php if missing (your canonical settings)
ORIG_INIT=$'set -ex\nmkdir -p /pvc/config /pvc/i18n \\\n         /var/www/html/gibbon/uploads/tmp /var/www/html/gibbon/uploads/cache\n[ -s /pvc/.gibbon_guid ] || (cat /proc/sys/kernel/random/uuid 2>/dev/null || date +%s%N) > /pvc/.gibbon_guid\nif [ ! -s /pvc/config/config.php ]; then\n  printf %s\\\\n \\\n    \\'<?php\\' \\\n    \\'$databaseServer   = getenv(\"DB_HOST\");\\' \\\n    \\'$databaseUsername = getenv(\"DB_USER\");\\' \\\n    \\'$databasePassword = getenv(\"DB_PASSWORD\");\\' \\\n    \\'$databaseName     = getenv(\"DB_NAME\");\\' \\\n    \\'$guid = getenv(\"GIBBON_GUID\");\\' \\\n    \\'if (!$guid) {\\' \\\n    \\'  $guidFile = __DIR__ . \"/../.gibbon_guid\";\\' \\\n    \\'  if (is_readable($guidFile)) { $guid = trim(file_get_contents($guidFile)); }\\' \\\n    \\' }\\' \\\n    \\'$caching     = 10;\\' \\\n    \\'$absoluteURL = getenv(\"ABSOLUTE_URL\");\\' \\\n  > /pvc/config/config.php\n  chown 33:33 /pvc/config/config.php || true\n  chmod 0660 /pvc/config/config.php || true\nfi\nchown -R 33:33 /var/www/html/gibbon/uploads || true\nchmod -R 0777 /var/www/html/gibbon/uploads || true\n'
kubectl -n "$NS" patch deploy/$DEP --type=json -p="$(
  python3 - <<'PY'
import json, os
script = os.environ["ORIG_INIT"]
print(json.dumps([
  {"op":"replace","path":"/spec/template/spec/initContainers/0/args/0","value":script}
]))
PY
)"

echo "== 7b) Roll the deployment =="
kubectl -n "$NS" rollout restart deploy/$DEP
kubectl -n "$NS" rollout status  deploy/$DEP --timeout=300s

echo "== 8) Cleanup helper pod =="
kubectl -n "$NS" delete pod pvc-tool --ignore-not-found

echo "All done ✅"

## 5. restore database 
- create new database if not avalable
### Steps

- from k8s control plane
- ensure file exist 
ls -lh /root/gibbon_2025_8_14.sql
set -eu
-  Vars
NS="gibbon-dev-deploy"
DB_NAME="gibbon"
DB_USER="gibbonuser"
DB_PASS="gibbonpass123"

- Stop the app so it doesn't write during restore
kubectl -n "$NS" scale deploy/gibbon-dev-app --replicas=0
kubectl -n "$NS" wait --for=delete pod -l app=gibbon-dev --timeout=180s

- Find MySQL pod
MYSQL_POD=$(kubectl -n "$NS" get pod -l app=gibbon-dev-mysql \
  -o jsonpath='{.items[0].metadata.name}')
echo "MYSQL_POD=$MYSQL_POD"

 1) Drop & recreate the DB using the app user
kubectl -n "$NS" exec -it "$MYSQL_POD" -c mysql -- bash -lc "
  set -eu
  export MYSQL_PWD='$DB_PASS'
  mysql -h 127.0.0.1 -P 3306 -u '$DB_USER' -e \"
    DROP DATABASE IF EXISTS \\\`$DB_NAME\\\`;
    CREATE DATABASE \\\`$DB_NAME\\\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  \"
"

 2) Copy dump into the MySQL pod and import
kubectl -n "$NS" cp /root/gibbon_2025_8_14.sql "$MYSQL_POD":/tmp/gibbon_2025_8_14.sql -c mysql

kubectl -n "$NS" exec -it "$MYSQL_POD" -c mysql -- bash -lc "
  set -eu
  export MYSQL_PWD='$DB_PASS'
  mysql --default-character-set=utf8mb4 \
    -h 127.0.0.1 -P 3306 -u '$DB_USER' '$DB_NAME' < /tmp/gibbon_2025_8_14.sql
  mysql -h 127.0.0.1 -P 3306 -u '$DB_USER' '$DB_NAME' -e \"
    SHOW TABLES;
    SELECT COUNT(*) AS people FROM gibbonPerson;
  \"
"

 3) Bring the app back
kubectl -n "$NS" scale deploy/gibbon-dev-app --replicas=1
kubectl -n "$NS" rollout status deploy/gibbon-dev-app --timeout=300s

- IF drop fails. Check the privileges your app user has:
kubectl -n "$NS" exec -it "$MYSQL_POD" -c mysql -- bash -lc "
  export MYSQL_PWD='$DB_PASS'
  mysql -h 127.0.0.1 -P 3306 -u '$DB_USER' -e \"SHOW GRANTS FOR '$DB_USER'@'%';\"
"
GRANT ALL PRIVILEGES ON `gibbon`.* TO 'gibbonuser'@'%';
FLUSH PRIVILEGES;

## 6. Connect to database within k8s from outside
### Permanent connection survive all roll out
****** this method will expose the mysql server. MUST have IP block or any other way to migrate the exposua
- create an overlay file for ingress-tcp
cat >/tmp/ingress-tcp-overlay.yaml <<'YAML'
controller:
  
  extraPorts:
    - name: tcp-3306
      containerPort: 3306
      hostPort: 3306
      protocol: TCP
    - name: tcp-3307
      containerPort: 3307
      hostPort: 3307
      protocol: TCP
    - name: tcp-3308
      containerPort: 3308
      hostPort: 3308
      protocol: TCP
    - name: tcp-3309
      containerPort: 3309
      hostPort: 3309
      protocol: TCP
tcp:
  "3306": "gibbon-dev-deploy/gibbon-dev-mysql:3306"
  "3307": "gibbon-demo-deploy/gibbon-demo-mysql:3306"
  "3308": "gibbon-banthach-deploy/gibbon-banthach-mysql:3306"
  "3309": "gibbon-thachcham-deploy/gibbon-thachcham-mysql:3306"
YAML

Apply it on top of your saved values:
helm -n ingress-nginx upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  -f /tmp/ingress-values.yaml \
  -f /tmp/ingress-tcp-overlay.yaml

kubectl -n ingress-nginx get cm | grep tcp   # should show ingress-nginx-tcp
kubectl -n ingress-nginx get ds ingress-nginx-controller -o jsonpath='{.spec.template.spec.containers[0].args}'
kubectl -n ingress-nginx get ds ingress-nginx-controller -o jsonpath='{.spec.template.spec.containers[0].ports}'
- quick check 
-- DNS/Ingress back up
curl -I https://gibbon-dev.vanhoa.edu.vn

-- TCP mapping live from allowed IP
nc -vz 5.78.68.173 3306

# Trouble shoot
## loging issue
NS=gibbon-dev-deploy
APP_POD=$(kubectl -n "$NS" get pods -l app=gibbon-dev \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}' \
  | tr " " "\n" | head -n1)

 1) Force PHP to log to a file in the PVC-backed uploads dir
kubectl -n "$NS" exec "$APP_POD" -c gibbon -- sh -lc '
set -eu
mkdir -p /var/www/html/gibbon/uploads/logs
chown -R www-data:www-data /var/www/html/gibbon/uploads/logs
cat > /usr/local/etc/php/conf.d/99-debug.ini <<EOF
display_errors=On
error_reporting=E_ALL
log_errors=On
error_log=/var/www/html/gibbon/uploads/logs/php-error.log
EOF
 2) Reload Apache IN PLACE (no pod restart) so php.ini is re-read
apache2ctl -k graceful || true
- show what PHP is using now
php -i | grep -E "error_log|display_errors|log_errors"
'

 3) In your browser, reproduce the Finance page error ONCE.

 4) Read the log from the same pod (no -it, so it won’t “hang”)
kubectl -n "$NS" exec "$APP_POD" -c gibbon -- sh -lc '
ls -lah /var/www/html/gibbon/uploads/logs || true
tail -n 200 /var/www/html/gibbon/uploads/logs/php-error.log || true
'
## Styling missing issue
0) Vars
NS=gibbon-thachcham-deploy

1) Find the MySQL pod
MYSQL_POD=$(kubectl -n "$NS" get pod -l app=gibbon-thachcham-mysql \
  -o jsonpath='{.items[0].metadata.name}')

2) Inspect the current value in gibbonSetting
kubectl -n "$NS" exec -it "$MYSQL_POD" -c mysql -- bash -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -D gibbon -e "
  SELECT scope,name,value
  FROM gibbonSetting
  WHERE name IN (\"absoluteURL\") OR value LIKE \"http://thachcham.%\";
"'

3) Force it to HTTPS
kubectl -n "$NS" exec -it "$MYSQL_POD" -c mysql -- bash -lc '
mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -D gibbon -e "
  UPDATE gibbonSetting
     SET value = \"https://thachcham.vanhoa.edu.vn\"
   WHERE scope = \"System\" AND name = \"absoluteURL\";
  SELECT scope,name,value FROM gibbonSetting WHERE name=\"absoluteURL\";
"'

4) Clear app cache & restart the app
APP=$(kubectl -n "$NS" get pod -l app=gibbon-thachcham \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}' | awk "NR==1{print}")
kubectl -n "$NS" exec "$APP" -c gibbon -- sh -lc 'rm -rf /var/www/html/gibbon/uploads/cache/* || true'
kubectl -n "$NS" rollout restart deploy/gibbon-thachcham-app
kubectl -n "$NS" rollout status  deploy/gibbon-thachcham-app --timeout=300s

5) Re-fetch HTML and confirm links are now HTTPS
APP=$(kubectl -n "$NS" get pod -l app=gibbon-thachcham \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}' | awk "NR==1{print}")
kubectl -n "$NS" exec "$APP" -c gibbon -- sh -lc \
  'curl -s -H "Host: thachcham.vanhoa.edu.vn" http://127.0.0.1/ > /tmp/index.html && \
   echo "Any http:// refs? (should be 0)"; \
   grep -o "http://thachcham.vanhoa.edu.vn" /tmp/index.html | wc -l && \
   echo "Sample CSS:" && \
   grep -inE "<link[^>]*(stylesheet|text/css)" /tmp/index.html | head -n 5'

## checking open ai key 
- checking if the openai key token in secret is matching with the key used in pod
NS="gibbon-dev-deploy"
SEC="gibbon-dev-openai"
APP_LABEL="app=gibbon-dev"

echo "[1] Secret (masked):"
SEC_MASKED=$(kubectl -n "$NS" get secret "$SEC" -o jsonpath='{.data.OPENAI_API_KEY}' \
  | base64 -d | awk '{print substr($0,1,6)"…"(length>4?substr($0,length-3):$0)}')
echo "    $SEC_MASKED"

SEC_LAST4=$(echo "$SEC_MASKED" | awk -F'…' '{print substr($2,length($2)-3)}')

echo "[2] Pods using the secret:"
PODS=$(kubectl -n "$NS" get pod -l "$APP_LABEL" \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}')
for p in $PODS; do
  echo " - $p"
  echo "   mounted file last4: \c"
  kubectl -n "$NS" exec "$p" -c gibbon -- sh -lc 'tail -c 4 /run/secrets/openai/OPENAI_API_KEY 2>/dev/null || echo MISSING'

  echo "   PHP getApiKey() last4: \c"
  kubectl -n "$NS" exec "$p" -c gibbon -- sh -lc '
    cat >/var/www/html/gibbon/_whichkey.php <<'"'"'PHP'"'"'
    <?php
      require __DIR__."/openai.php";
      $k = function_exists("getApiKey") ? getApiKey(false) : null;
      echo $k ? substr($k,-4) : "MISSING";
    PHP
    php -d display_errors=0 /var/www/html/gibbon/_whichkey.php 2>/dev/null || true
    rm -f /var/www/html/gibbon/_whichkey.php 2>/dev/null || true
    echo
  '
done

echo "[3] Expect last4 = $SEC_LAST4"


# *************** NUKE DEPLOYMENT - PROCEED WITH CAUTION - IRREVERSIBLE****************

## Back up database, files if needed 
## Back up database, files if needed (AGAIN)