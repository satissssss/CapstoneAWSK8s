**Steps done on Capstone project**
1. Setup new deployment EC2 server
2. Setup All below softwares
   Docker
   Kubernetes
   EKSCTL
   Git
   Node
   Helm
3. Containerize Apps
4. Create EKS cluster with nodes
5. Deploy apps with database into EKS
6. Test blue green deployment

**Folders purpose on GIT:**
HPA - Contains k8s config file to test the HPA
database-initializer - Contains the Docker project for DB initializer 
ekscluster - contains the EKScluster config
events-api - Contains the Docker project for API
events-website - Contains the Docker projet for Website
k8s-config - Contains different k8s config files.


**Commands used to achieve the above**
Setup Access Key and secret:

Install Kubernetes and provide permissions:
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl

chmod +x ./kubectl 

mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

Check kubectl installed:
kubectl

Set the following variables:
export ARCH=amd64
export PLATFORM=$(uname -s)_$ARCH

Install EKS and provide permissions:
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin

Check eksctl installed:
eksctl

Install Docker and start:
sudo yum install docker

sudo systemctl enable docker
sudo systemctl start docker


Check docker installed:
docker

Run this during startup of EC2:
sudo systemctl enable docker (set to start service on boot)
sudo systemctl start docker (start the service)

Install Git:
sudo yum install git

Install Node:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts

Check node installed:
npm

Install Helm and permissions:
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

Check helm installed:
helm 


Pull git repo having all config files:
mkdir capstoneapp
cd capstoneapp
git init
git pull https://github.com/satissssss/CapstoneAWSK8s.git

Install events-api and run:
cd events-api
npm install
node server.js


Install events-web and run:
cd ~/eventsapp/events-website/
npm install
npm start

App running on public DNS:
https://ec2-54-211-29-128.compute-1.amazonaws.com/


Create Dockerfile and .dockerignore for both api and website:

Provide permission for docker:
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user

Create container image:
docker build . -t events-api:v1.0
docker build . -t events-website:v1.0


Run Container images:
docker run -d -p 8082:8082 events-api:v1.0
docker run -d -p 8080:8080 -e SERVER='http://localhost:8082' --network="host" events-website:v1.0

App running on public DNS:
http://ec2-54-211-29-128.compute-1.amazonaws.com:8080/

Upload image to ECR:
docker login -u AWS -p $(aws ecr get-login-password --region us-east-1) 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-api

docker build -t events-api .
docker tag events-api:latest 065145552369.dkr.ecr.us-east-1.amazonaws.com/cp-events-api
docker push 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-api:latest


docker login -u AWS -p $(aws ecr get-login-password --region us-east-1)  065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website

docker build -t events-website .
docker tag events-website:latest  065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website
docker push  065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website:latest

Run container from ECR image:
docker run -d -p 8082:8082 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-api:latest

docker run -d -p 8080:8080 -e SERVER='http://localhost:8082' --network="host" 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website:latest

Check App running:
http://ec2-54-211-29-128.compute-1.amazonaws.com:8080/

Update version:
Update source and create new image version
docker build -t events-website .
docker tag events-website:latest  065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website:2.0
docker push  065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website:2.0

docker run -d -p 8082:8082 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-api:1.0
docker run -d -p 8080:8080 -e SERVER='http://localhost:8082' --network="host" 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-website:2.0

Check Updated version running:
http://ec2-54-211-29-128.compute-1.amazonaws.com:8080/

Setup EKS Cluster:
eksctl create cluster -f cluster-config.yaml

update kubectl with cluster created
aws eks update-kubeconfig --region us-east-1 --name <<Cluster name>>


Check cluster and nodes created:
kubectl cluster-info
kubectl get services 
kubectl get nodes
kubectl get pods

Deploy Mario app on EKS:
cd ~/eventsapp
mkdir kubernetes-config

kubectl apply -f mario-pod.yaml
kubectl get pods
kubectl describe pod supermario-pod


Expose with loadbalancer:
kubectl expose pod supermario-pod-capstone --type=LoadBalancer --name=supermario-svc --port=80 --target-port=8080
kubectl get service

Check logs of pod:
kubectl logs supermario-pod

Check the deployed App in external IP created in Loadbalancer:
http://a157b485dc6b747e2a24e27710f33549-115759024.us-east-1.elb.amazonaws.com/

Install DB through Helm:
helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb

Check created DB
helm list
helm status database-server

Setup ECR with events-job repo
docker login -u AWS -p $(aws ecr get-login-password --region us-east-1) 065145552369.dkr.ecr.us-east-2.amazonaws.com/events-job

docker build . -t 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-job:v1.0
docker push 065145552369.dkr.ecr.us-east-1.amazonaws.com/events-job:v1.0

Check secrets created:
kubectl get secrets

Setup cronjob for Maria DB initialize with records with image created in ECR:
Run the following command to deploy the job:
kubectl apply -f db_init_job.yaml

Check Job status:
kubectl get jobs -w

View the logs:
kubectl get pods
kubectl logs mariadb-init-capstone


Create new deployments:
kubectl apply -f api-deployment.yaml 
kubectl apply -f web-deployment.yaml 

Verify it was deployed:
kubectl get deploy

Create new services:
kubectl apply -f api-service.yaml 
kubectl apply -f web-service.yaml 

Verify it was deployed:
kubectl get service

Verify the Public dns of web is working:
http://a9bca30b8f2b343a2830297c2695188c-715389031.us-east-1.elb.amazonaws.com/

Setup cronjob and verify
kubectl apply -f cronjob Capstone-cronjob

Check Blue/green deploy:
kubectl apply -f web-deployment-v2.yaml
kubetcl apply -f web-service.yaml

Delete older deployment:
kubectl delete -f web-deployment.yaml

Test HPA:
kubectl apply -f HPAdeployment.yaml 
kubectl apply -f autoscale-hpa.yaml 

Run new app to use memory:
kubectl apply -f HPAservice.yaml

Call node App in loop:
while true; do curl http://a9b673c5f1a674d0c8d8932afc7a56ea-614781299.us-east-1.elb.amazonaws.com/; done;

Check hpa:
kubectl get hpa

Setup cronjob:
kubectl apply -f caps-cronjob.yaml 

Check cronjob running:
kubectl apply -f cronjob-caps.yaml 

Delete EKS:
eksctl delete cluster -f cluster-config.yaml

Docker stopped and removed running instance

