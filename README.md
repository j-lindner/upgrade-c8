# Example Upgrade path for Camunda 8.2 -> 8.5


# Install Camunda 8.2.20 with Elasticsearch 7.17.15
```bash
helm install --namespace camunda camunda camunda/camunda-platform -f ./Camunda8.2.20-with-ES7.17.15-values.yaml --version 8.2.20
```

## Create sample data
via process-solution-template-sample-data
```bash
curl -X POST http://localhost:8080/process/start -H "Content-Type: application/json" -d '{"businessKey": "2302"}'
```

# Upgrade Camunda 8.2.20 to 8.3.3
Docs: https://docs.camunda.io/docs/self-managed/operational-guides/update-guide/820-to-830/

## Update deprecated Elasticsearch properties
```bash
kubectl set env statefulset/elasticsearch-master node.roles='data','ingest','master','remote_cluster_client','ml'

kubectl set env statefulset/elasticsearch-master node.data-
kubectl set env statefulset/elasticsearch-master node.ingest-
kubectl set env statefulset/elasticsearch-master node.master-
kubectl set env statefulset/elasticsearch-master node.remote_cluster_client-
kubectl set env statefulset/elasticsearch-master node.ml-
```

## Data Retention in ES
Option Two: Update PVs manually, see https://docs.camunda.io/docs/self-managed/platform-deployment/helm-kubernetes/upgrade/#elasticsearch---data-retention

Patch PVs, so they can later be re-used
```bash
kubectl get pv -o json | jq -r '.items[].metadata.name' | xargs -I {} kubectl patch pv {} -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

```bash
kubectl scale statefulset elasticsearch-master --replicas 0
```

```bash
kubectl delete pvc elasticsearch-master-elasticsearch-master-0
kubectl delete pvc elasticsearch-master-elasticsearch-master-1
```

Make PVs bind to new Elastic PVCs
```bash
kubectl get pv -o yaml | yq -r '.items[] | select(.spec.claimRef != null) | select(.spec.claimRef.name == "elasticsearch-master-elasticsearch-master-0")' > master-0.yaml
yq eval 'del(.metadata.resourceVersion, .spec.claimRef.resourceVersion, .spec.claimRef.uuid, .spec.claimRef.name)' -i master-0.yaml
yq eval '.spec.claimRef.name = "data-camunda-elasticsearch-master-0"' -i master-0.yaml
kubectl apply -f master-0.yaml
rm master-0.yaml
```

```bash
kubectl get pv -o yaml | yq -r '.items[] | select(.spec.claimRef != null) | select(.spec.claimRef.name == "elasticsearch-master-elasticsearch-master-1")' > master-1.yaml
yq eval 'del(.metadata.resourceVersion, .spec.claimRef.resourceVersion, .spec.claimRef.uuid, .spec.claimRef.name)' -i master-1.yaml
yq eval '.spec.claimRef.name = "data-camunda-elasticsearch-master-1"' -i master-1.yaml
kubectl apply -f master-1.yaml
rm master-1.yaml
```

## actual upgrade

```bash
kubectl -n camunda delete deployment camunda-operate
kubectl -n camunda delete deployment camunda-tasklist
kubectl -n camunda delete deployment camunda-zeebe-gateway
kubectl -n camunda delete statefulset camunda-zeebe
```

```bash
helm upgrade -f ./Camunda8.3.3-with-ES8.8.2-values.yaml camunda camunda/camunda-platform --version 8.3.3
```


# Upgrade Camunda 8.3.3 to 8.4.0
Docs: https://docs.camunda.io/docs/self-managed/operational-guides/update-guide/830-to-840/

```bash
helm upgrade -f ./Camunda8.4.0-with-ES8.9.2-values.yaml  camunda camunda/camunda-platform --version 9.3.4
```


# Upgrade Camunda 8.4.0 to 8.5.
Docs: https://docs.camunda.io/docs/self-managed/operational-guides/update-guide/840-to-850/
Docs: https://docs.camunda.io/docs/self-managed/setup/upgrade/#from-camunda-84-to-85

```bash
kubectl -n camunda delete -l app.kubernetes.io/name=identity deployment
```

delete old ingress, so "/" will be available for new grpc-specific Ingress
```bash
kubectl delete ingress camunda-zeebe-gateway
```

```bash
helm upgrade -f ./Camunda8.5.0-with-ES8.13.0-values.yaml  camunda camunda/camunda-platform --version 10.0.2
```
