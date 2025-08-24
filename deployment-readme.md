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
key rotation
kubectl -n "$NS" delete secret gibbon-dev-openai
kubectl -n "$NS" create secret generic gibbon-dev-openai \
  --from-literal=OPENAI_API_KEY='sk-...new...' 

- if deployment is running
kubectl -n "$NS" rollout restart deployment/gibbon-dev-app

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

## 5. restore database 
- create new database if not avalable
### Steps

- from k8s control plane
- ensure file exist 
ls -lh /root/gibbon_2025_8_14.sql
set -eu
DB_HOST="gibbon-dev-mysql.gibbon-dev-deploy.svc.cluster.local"
DB_NAME="gibbon"
DB_USER="gibbonuser"
DB_PASS="gibbonpass123"
DB_PORT=3306
MYSQL_ROOT_PASSWORD="gibbonpass123"

NS=gibbon-dev-deploy
MYSQL_POD=$(kubectl -n "$NS" get pod -l app=gibbon-dev-mysql \
  -o jsonpath='{.items[0].metadata.name}')
echo "MYSQL_POD=${MYSQL_POD}"
kubectl -n "$NS" exec -it "$MYSQL_POD" -c mysql -- bash -lc '
set -eu
mysql -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
  CREATE DATABASE IF NOT EXISTS \`$DB_NAME\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  CREATE USER IF NOT EXISTS \"$DB_USER\"@\"%\" IDENTIFIED BY \"$DB_PASS\";
  ALTER USER \"$DB_USER\"@\"%\" IDENTIFIED BY \"$DB_PASS\";
  GRANT ALL PRIVILEGES ON \`gibbon\`.* TO \"$DB_USER\"@\"%\";
  FLUSH PRIVILEGES;
  SELECT user,host FROM mysql.user WHERE user=\"$DB_USER\";
"'

- verify
APP_POD=$(kubectl -n "$NS" get pods -l app=gibbon-dev \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}' | awk "NR==1{print}")
kubectl -n "$NS" exec -it "$APP_POD" -c gibbon -- bash -lc '
DB_HOST="gibbon-dev-mysql.gibbon-dev-deploy.svc.cluster.local"
DB_NAME="gibbon"
DB_USER="gibbonuser"
DB_PASS="gibbonpass123"
DB_PORT=3306
MYSQL_PWD="$DB_PASS" mysql -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" -e "SELECT 1;"
'

- copy dump file and restore dump
kubectl -n "$NS" cp /root/gibbon_2025_8_14.sql "$APP_POD":/root/gibbon_2025_8_14.sql -c gibbon


kubectl -n "$NS" exec -it "$APP_POD" -c gibbon -- bash -lc '
DB_HOST="gibbon-dev-mysql.gibbon-dev-deploy.svc.cluster.local"
DB_NAME="gibbon"
DB_USER="gibbonuser"
DB_PASS="gibbonpass123"
DB_PORT=3306
MYSQL_PWD="$DB_PASS" mysql --default-character-set=utf8mb4 \
  -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" "$DB_NAME" < /root/gibbon_2025_8_14.sql
MYSQL_PWD="$DB_PASS" mysql -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" "$DB_NAME" \
  -e "SHOW TABLES; SELECT COUNT(*) AS people FROM gibbonPerson;"
'
