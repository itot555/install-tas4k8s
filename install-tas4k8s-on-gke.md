## provisioning GKE cluster
```
gcloud container clusters create demo \
 --image-type ubuntu --cluster-version 1.16.9-gke.6 \ 
 --machine-type e2-standard-2 \
 --zone asia-northeast1-b \
 --preemptible --num-nodes 6
```

## login steplin

## dowonload tas4k8s
```
cd workspace
mkdir tas4k8s
cd tas4k8s
API_TOKEN=****************
pivnet login --api-token=${API_TOKEN}
pivnet download-product-files --product-slug='tas-for-kubernetes' --release-version='0.2.0' --product-file-id=696152
tar xvf tanzu-application-service.0.2.0-build.104.tar
cd tanzu-application-service
```

## install ytt, kapp, kbld
```
sudo su -
wget -O- https://k14s.io/install.sh | bash
kapp --version
kbld --version
ytt version
```

## configuring default storageclass 
```
gcloud container clusters get-credentials demo --zone asia-northeast1-b
kubectl get storageclasses.storage.k8s.io
```

## generate configuration values
```
mkdir configuration-values
cd tanzu-application-service
./bin/generate-values.sh -d "tas4k8s.dx-make-it-real.net" > ../configuration-values/deployment-values.yml
```

## (Optional) Use Load Balancer Service for Ingress Gateway
```
cd ..
cd configuration-values
vim load-balancer-values.yml
```

## Configure System Registry Values
```
export TANZU_NETWORK_USERNAME=****************
export TANZU_NETWORK_PASSWORD=****************
cat <<EOF > system-registry-values.yml
#@data/values
---
system_registry:
  hostname: registry.pivotal.io
  username: ${TANZU_NETWORK_USERNAME}
  password: ${TANZU_NETWORK_PASSWORD}
EOF
```

## Configuring Application Image Registry
```
GCP_PROJECT_ID=$(gcloud config get-value core/project)
gcloud config list
gcloud iam service-accounts create tas-app-images-push-pull
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
 --member serviceAccount:tas-app-images-push-pull@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
 --role roles/storage.admin
gcloud iam service-accounts keys create \
 --iam-account "tas-app-images-push-pull@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
 gcr-storage-admin.json
```

## Creating the Configuration File
```
cat << YAML >app-registry.yaml
 #@data/values
 ---
 app_registry:
   hostname: asia.gcr.io
   repository: asia.gcr.io/$(gcloud config get-value project)/tas-app-images
   username: _json_key
   password: |
 $(cat gcr-storage-admin.json | sed 's/^/    /g')
 YAML
```

## Install TAS4K8s on GKE
```
gcloud container clusters get-credentials demo-tas4k8s --zone asia-northeast1-b
./bin/install-tas.sh ../configuration-values
```

