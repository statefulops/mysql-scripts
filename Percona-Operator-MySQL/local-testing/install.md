# Local testing on kind

Notes for bringing up a PXC cluster on a local [kind](https://kind.sigs.k8s.io/)
cluster with a MinIO-backed S3 bucket, then exercising backups and binlog
collection.

## 1. Create the kind cluster

Creates the kind cluster plus a local MinIO S3 endpoint on `localhost:9000`.

```bash
curl -fsSL https://raw.githubusercontent.com/statefulops/kind-scripts/main/s3/create-cluster.sh | bash
```

## 2. Install the operator

Add the Percona Helm repo:

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update
```

Safety check — only ever run this against a local kind cluster:

```bash
CURRENT_CONTEXT=$(kubectl config current-context)
case "$CURRENT_CONTEXT" in
  *kind*) echo "OK: using kind context '$CURRENT_CONTEXT'" ;;
  *) echo "ABORT: context '$CURRENT_CONTEXT' is not a kind cluster"; exit 1 ;;
esac
```

> Inside an interactive shell, use `return 1` instead of `exit 1` so a failed
> check does not close the terminal.

Install the operator and switch to its namespace:

```bash
kubectl create namespace ps-operator
helm install ps-operator percona/ps-operator --namespace ps-operator

kubectl config set-context --current --namespace=ps-operator
```

=-====

## 3. Deploy the cluster

```bash
kubectl apply -f cr.yaml
 
```

## 4. Connect to the cluster

Get the root password:

```bash
export NAMESPACE=ps-operator
export CLUSTER_NAME=ps-cluster1
export SECRET_NAME=${CLUSTER_NAME}-psuser-root

kubectl get secret "$SECRET_NAME" -n "$NAMESPACE" \
 -o jsonpath='{.data.user}' | base64 --decode && echo 
 
kubectl get secret "$SECRET_NAME" -n "$NAMESPACE" -o jsonpath='{.data.password}' | base64 --decode && echo

```

Launch a client pod and connect:

```bash
kubectl run -n $NAMESPACE -i --rm --tty percona-client \
--image=percona/percona-server:8.4 --restart=Never -- bash -il
```

Inside the client pod:

```bash
mysql -u root -p -h ps-cluster1-haproxy
```


## 4. Take a backup

kubectl get secret rustfs-admin-credentials -n rustfs -o jsonpath='{.data.secret-key}'



```bash
kubectl apply -f backup-secret-s3.yaml   # secret/my-cluster-name-backup-s3 created
kubectl apply -f backup.yaml
```

Follow the backup job's logs (substitute your own pod name):

```bash
kubectl logs xb-backup1-5l74d
```

```text
...
+ GARBD_EXIT_CODE=0
+ case ${GARBD_EXIT_CODE} in
+ log INFO 'Backup was finished successfully'
+ exit 0
2026-09-18 08:44:17 [INFO] Backup was finished successfully
```

## 5. Verify the backup in MinIO

```bash
export AWS_ACCESS_KEY_ID=myadminuser
export AWS_SECRET_ACCESS_KEY=$(kubectl get secret rustfs-admin-credentials -n rustfs -o jsonpath='{.data.secret-key}' | base64 -d)
export AWS_DEFAULT_REGION=us-east-1

aws --endpoint-url http://localhost:9000 s3 ls my-kind-bucket/
```

## 6. Enable binlog collection

```bash
kubectl apply -f cr-binlogs.yaml
kubectl get po
```

A `-pitr-` pod appears once binlog collection is running:

```text
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
