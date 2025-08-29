



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
0) Vars
NS=gibbon-dev-deploy
DEP=gibbon-dev-app
PVC=gibbon-dev-uploads-pvc

1) Scale app down (no writers)
kubectl -n "$NS" scale deploy/$DEP --replicas=0
kubectl -n "$NS" wait --for=delete pod -l app=gibbon-dev --timeout=180s

2) Start a tiny helper pod that mounts the same PVC
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

3) Delete the file from the PVC
kubectl -n "$NS" exec pvc-tool -- sh -lc 'rm -f /pvc/config/config.php && ls -l /pvc/config || true'

4) Temporarily change the Deployment so it doesn’t require or recreate config.php

Remove the config.php volumeMount (it’s the 2nd mount on the container in your YAML).

Disable the part of the init-container that auto-creates config.php.

4a) remove the config.php volumeMount
kubectl -n "$NS" patch deploy/$DEP --type=json -p='[
 {"op":"remove","path":"/spec/template/spec/containers/0/volumeMounts/1"}
]'

4b) replace init script so it skips creating config.php
kubectl -n "$NS" patch deploy/$DEP --type=json -p="$(
  python3 - <<'PY'
import json,sys
script = r'''set -ex
mkdir -p /pvc/config /pvc/i18n /var/www/html/gibbon/uploads/tmp /var/www/html/gibbon/uploads/cache
- (installer mode) DO NOT create /pvc/config/config.php here
chown -R 33:33 /var/www/html/gibbon/uploads || true
chmod -R 0777 /var/www/html/gibbon/uploads || true
'''
print(json.dumps([
  {"op":"replace","path":"/spec/template/spec/initContainers/0/args/0","value":script}
]))
PY
)"

5) Scale up and run the installer
kubectl -n "$NS" scale deploy/$DEP --replicas=1
kubectl -n "$NS" rollout status deploy/$DEP --timeout=300s

Now open https://gibbon-dev.vanhoa.edu.vn/installer/ and complete the steps.
The installer will write /var/www/html/gibbon/config.php inside the pod filesystem.

6) Persist that config.php back into the PVC
APP_POD=$(kubectl -n "$NS" get pod -l app=gibbon-dev -o jsonpath='{.items[0].metadata.name}')

- copy file contents from app pod into PVC via the helper
kubectl -n "$NS" exec "$APP_POD" -c gibbon -- sh -lc 'cat /var/www/html/gibbon/config.php' \
 | kubectl -n "$NS" exec pvc-tool -- sh -lc 'mkdir -p /pvc/config && cat > /pvc/config/config.php && chown 33:33 /pvc/config/config.php && chmod 660 /pvc/config/config.php'

7) Restore your Deployment to “normal”

Easiest: re-apply your canonical YAML (the one in Git/Jenkins) so the original init script and the config.php volumeMount come back:

kubectl -n "$NS" apply -f k8s/gibbon-deployment.yaml
kubectl -n "$NS" rollout status deploy/$DEP --timeout=300s

(If you prefer patches instead of re-applying YAML, you can add the volumeMount back and replace the init args with the original block, but re-applying your file is simpler and source-of-truth-friendly.)

8) Clean up helper
kubectl -n "$NS" delete pod pvc-tool --ignore-not-found

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
