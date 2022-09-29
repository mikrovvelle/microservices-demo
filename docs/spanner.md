# Cloud Spanner + OnlineBoutique

This guide contains instructions for overriding the default in-cluster `redis` database for `cartservice` with Cloud Spanner.

Run these steps after the **Create a GKE cluster** step and before the **Deploy the sample app to the cluster** in the README Quickstart (GKE).

## Steps to use Spanner for your cluster

All commands below should be run from the root directory of this project.
Run the commands in the same session to keep environmental variables consistent.

### 0. Environmental Variables

Choose your desired project, region, and zone.

```sh
PROJECT_ID="<your-project-id>"
REGION="<your-GCP-region>"      # e.g. us-central1
ZONE="<your-GCP-zone>"          # e.g. us-central1-b
```

### 1. Create a GKE cluster with Workload Identity enabled

These commands are similar as what is listed in the README Quickstart (GKE),
but include a flag to [enable workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#enable).
This will allow the GKE service account to query Spanner.

GKE autopilot mode (see [Autopilot
overview](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)
to learn more):

```sh
gcloud container clusters create-auto onlineboutique \
    --project=${PROJECT_ID} --region=${REGION}
```

GKE Standard mode:

```sh
gcloud container clusters create onlineboutique \
    --project=${PROJECT_ID} --zone=${ZONE} \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --machine-type=n2-standard-2 --num-nodes=4
```

### 2. Enable Cloud Spanner on your project

```sh
gcloud services enable spanner.googleapis.com --project=${PROJECT_ID}
```

### 3. Create a Cloud Spanner instance, database, and table

Note: See the documentation to list [available Spanner configuration names](https://cloud.google.com/spanner/docs/getting-started/set-up#run_the_gcloud_tool), or run `gcloud spanner instance-configs list`

```sh
SPANNER_REGION_CONFIG="<your-spanner-region-config-name>"
# e.g. "regional-us-central1"

gcloud spanner instances create hipstershop \
    --description="online boutique backend" \
    --config="${SPANNER_REGION_CONFIG}" \
    --instance-type=free-instance

gcloud spanner databases create carts \
    --project=${PROJECT_ID} \
    --instance="hipstershop" \
    --database-dialect=GOOGLE_STANDARD_SQL \
    --ddl-file=./src/cartservice/ddl/CartItems.ddl
```

### 4. Grant the default GCE/GKE service account read/write permission to Spanner

```sh
# grant the default GKE service account databaseUser in Spanner
SVCACCT=$(gcloud iam service-accounts list --filter 'Compute Engine default' --format 'value(email)')
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:${SVCACCT}" \
    --role="roles/spanner.databaseUser"

# annotate the kubectl serviceAccountName with the gcloud svcaccount
kubectl annotate serviceaccount default \
    iam.gke.io/gcp-service-account=${SVCACCT}

# tell gcloud that the kubectl account maps to the gcloud one
gcloud iam service-accounts add-iam-policy-binding \
    "${SVCACCT}" --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:${PROJECT_ID}.svc.id.goog[default/default]"
```



### 5. Congigure the Kubernetes manifest to use Spanner for the `cartservice`.

```sh
cp ./release/kubernetes-manifests.yaml ./release/updated-manifests.yaml
sed -i "s/name: REDIS_ADDR/name: SPANNER_PROJECT/g" ./release/updated-manifests.yaml
sed -i "s/value: \"redis-cart:6379\"/value: \"${PROJECT_ID}\"/g" ./release/updated-manifests.yaml
```

### 6. Apply the updated manifest

```sh
kubectl apply -f ./release/updated-manifests.yaml
```

#### Optional: delete the unused redis-cart service

```sh
kubectl delete deployment redis-cart
```

### 7. Wait for the Pods to be ready

```sh
kubectl get pods
```

After a few minutes, you should see something like this:

```
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-76bdd69666-ckc5j               1/1     Running   0          2m58s
cartservice-66d497c6b7-dp5jr             1/1     Running   0          2m59s
checkoutservice-666c784bd6-4jd22         1/1     Running   0          3m1s
currencyservice-5d5d496984-4jmd7         1/1     Running   0          2m59s
emailservice-667457d9d6-75jcq            1/1     Running   0          3m2s
frontend-6b8d69b9fb-wjqdg                1/1     Running   0          3m1s
loadgenerator-665b5cd444-gwqdq           1/1     Running   0          3m
paymentservice-68596d6dd6-bf6bv          1/1     Running   0          3m
productcatalogservice-557d474574-888kr   1/1     Running   0          3m
recommendationservice-69c56b74d4-7z8r5   1/1     Running   0          3m1s
shippingservice-6ccc89f8fd-v686r         1/1     Running   0          2m58s
```

### 8. Access the web frontend in a browser

```sh
kubectl get service frontend-external | awk '{print $4}'
```

**Note**- you may see `<pending>` while GCP provisions the load balancer. If this happens, wait a few minutes and re-run the command.

### 9. Teardown

To stop the GKE cluster:

```sh
kubectl delete -f ./release/updated-manifests.yaml
```

To delete the GKE cluster, either:

```sh
gcloud container clusters delete onlineboutique --zone=${ZONE}
```

or

```sh
gcloud container clusters delete onlineboutique --region=${REGION}
```

To delete the Spanner instance:

```sh
gcloud spanner instances delete hipstershop
```