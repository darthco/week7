enable APIs
gcloud services enable container.googleapis.com \
    logging.googleapis.com \
    monitoring.googleapis.com \
    cloudtrace.googleapis.com

Now add telemetry, monitoring, cloudtrace code to the python file

Add requirements to requirements.txt

Add Dockerfile

Build Docker image

docker build -t demo_log .

Tag the image
docker tag demo_log us-central1-docker.pkg.dev/operating-edge-473204-j5/my-repo/demo_log:latest

push the image to artifact repository
docker push us-central1-docker.pkg.dev/operating-edge-473204-j5/my-repo/demo_log:latest

start gke cluster name - demo-log-ml-cluster
why this config - getting quota error so using less disk size and compute
gcloud container clusters create demo-log-ml-cluster \
  --zone=us-central1-a \
  --num-nodes=1 \
  --machine-type=e2-small \
  --disk-type=pd-standard \
  --disk-size=80 \
  --workload-pool=$(gcloud config get-value project).svc.id.goog \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM

create telemetry access account
gcloud iam service-accounts create telemetry-access \
    --display-name "Access for GKE ML service"

PROJECT_ID=$(gcloud config get-value project)

---- ---- ----
To login and give auth to VM -
gcloud auth login

OR maybe try this
sudo apt-get install google-cloud-cli-gke-gcloud-auth-plugin

gcloud config set container/use_application_default_credentials true

---- ---- ----

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:telemetry-access@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:telemetry-access@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudtrace.agent"

---- 3 steps for Kube ----

kubectl create serviceaccount telemetry-access --namespace default

kubectl annotate serviceaccount telemetry-access \
  --namespace default \
  iam.gke.io/gcp-service-account=telemetry-access@$PROJECT_ID.iam.gserviceaccount.com

gcloud iam service-accounts add-iam-policy-binding telemetry-access@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[default/telemetry-access]"

gcloud container clusters get-credentials demo-log-ml-cluster \
  --zone=us-central1-a
---- ----- ----

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml

To get the exposed service info:
kubectl get service demo-log-ml-service
kubectl get pods
kubectl logs demo-log-ml-service-5bcb66fb75-6n9wz --previous


if things dont work out, delete the workload and try again
kubectl delete deployment demo-log-ml-service


curl -X POST http://34.30.191.204:80/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2}'

sudo apt-get install wrk
wrk -t4 -c2000 -d200s --latency -s post.lua http://34.63.96.164:80/predict

# Git setup
git remote add origin https://github.com/darthco/21F2000513_IITMBS_MLOPS_OPPE1.git
git remote -v
git push -u origin master


kubectl get pods
kubectl get hpa
