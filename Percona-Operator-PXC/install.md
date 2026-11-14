# Percona XtraDB Cluster Operator — Install Notes

## 1. Create the kind cluster

```bash
curl -fsSL https://raw.githubusercontent.com/statefulops/kind-scripts/main/s3/create-cluster.sh | bash -s -- v1.37.0 xtradb
```

## 2. Install the operator

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

kubectl create namespace pxc-operator
helm install pxc-operator percona/pxc-operator --version 1.20.1 --namespace pxc-operator

kubectl config set-context --current --namespace=pxc-operator
```

## 3. Deploy the cluster

```bash
kubectl apply -f cr.yaml
```

Check status:

```bash
kubectl get perconaxtradbcluster
```

While initializing:

```
NAME       ENDPOINT                        STATUS         PXC   PROXYSQL   HAPROXY   AGE
cluster1   cluster1-haproxy.pxc-operator   initializing   1                3         3m16s
```

Once ready:

```
NAME       ENDPOINT                        STATUS   PXC   PROXYSQL   HAPROXY   AGE
cluster1   cluster1-haproxy.pxc-operator   ready    3                3         5m23s
```

__

## 4. Connect to the cluster

Get the root password:

```bash
kubectl get secret cluster1-secrets --template='{{.data.root | base64decode}}{{"\n"}}'
```

Launch a client pod and connect:

```bash
kubectl run -i --rm --tty percona-client --image=percona/percona-xtradb-cluster:8.4 --restart=Never -- bash -il
```

Inside the client pod:

```bash
mysql -u root -p -h cluster1-haproxy
```

```sql
mysql> select @@hostname
    -> ;
+----------------+
| @@hostname     |
+----------------+
| cluster1-pxc-0 |
+----------------+
1 row in set (0.00 sec)

mysql>
```
@TODO add overview etc

## 5. Backups ( RustFS)

Create the backup S3 secret from the RustFS admin credentials:

```bash
kubectl create secret generic my-cluster-name-backup-s3 --from-literal=AWS_ACCESS_KEY_ID="$(kubectl get secret rustfs-admin-credentials -n rustfs -o jsonpath='{.data.access-key}' | base64 -d)" --from-literal=AWS_SECRET_ACCESS_KEY="$(kubectl get secret rustfs-admin-credentials -n rustfs -o jsonpath='{.data.secret-key}' | base64 -d)" --dry-run=client -o yaml | kubectl apply -f -
```

Run a backup:

```bash
kubectl apply -f backup.yaml
```

Check backup logs:

```bash
kubectl logs xb-backup1-5l74d
```

```
...
+ GARBD_EXIT_CODE=0
+ case ${GARBD_EXIT_CODE} in
+ log INFO 'Backup was finished successfully'
+ exit 0
2026-09-18 08:44:17 [INFO] Backup was finished successfully
```

Verify the backup landed in S3:

```bash
export AWS_ACCESS_KEY_ID=myadminuser
export AWS_SECRET_ACCESS_KEY=$(kubectl get secret rustfs-admin-credentials -n rustfs -o jsonpath='{.data.secret-key}' | base64 -d)
export AWS_DEFAULT_REGION=us-east-1

aws --endpoint-url http://localhost:9000 s3 ls my-kind-bucket/
```



```
NAME                             READY   STATUS      RESTARTS   AGE
cluster1-haproxy-0               2/2     Running     0          24m
cluster1-haproxy-1               2/2     Running     0          22m
cluster1-haproxy-2               2/2     Running     0          21m
cluster1-pitr-77b9f9f8c5-dcfhf   1/1     Running     0          4m19s
cluster1-pxc-0                   3/3     Running     0          24m
cluster1-pxc-1                   3/3     Running     0          22m
cluster1-pxc-2                   3/3     Running     0          20m
pxc-operator-645854c986-65dvg    1/1     Running     0          25m
xb-backup1-5l74d                 0/1     Completed   0          13m
```
## 7 Restore

@todo do backup then pitr testing ?

disable any pitr

kubectl patch pxc cluster1  \
  --type merge \
  -p '{"spec":{"backup":{"pitr":{"enabled":false}}}}'


  cat <<EOF | kubectl apply -f-
apiVersion: "pxc.percona.com/v1"
kind: "PerconaXtraDBClusterRestore"
metadata:
  name: "restore1"
spec:
  pxcCluster: "cluster1"
  backupName: "backup1"
EOF

point in time 

cat <<EOF | kubectl apply -f-
apiVersion: "pxc.percona.com/v1"
kind: "PerconaXtraDBClusterRestore"
metadata:
  name: "restore1"
spec:
  pxcCluster: "cluster1"
  backupName: "backup1"
  pitr:
    type: date
    date: "2020-12-31 09:37:13"
    backupSource:
      storageName: "s3-rustfs"
EOF


## 6. Binlog collection

```bash
kubectl apply -f cr-binlogs.yaml
```

Verify pods:

```bash
kubectl get po
```