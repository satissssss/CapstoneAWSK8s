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

**Commands used to achieve the above**
Setup new deployment EC2:
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

Check testpod by running:
exec testpod -it --/bin/bash
eksctl utils describe-stacks --region=us-east-1 --cluster=sat-capstone-cluste

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
kubectl expose pod supermario-pod --type=LoadBalancer --name=supermario-svc --port=80 --target-port=8080
kubectl get service

Check logs of pod:
kubectl logs supermario-pod


Delete the pod:
kubectl delete pod supermario-pod 
kubectl get pods

Delete the service:
kubectl delete service supermario-svc
kubectl get svc

Create new deployment for API:
kubectl apply -f api-deployment.yaml 

Verify it was deployed:
kubectl get deployments
kubectl get pods

Describe pod:
kubectl describe pod <pod-name>

Create new deployment for Web:
kubectl apply -f web-deployment.yaml 

Expose website with loadbalancer:
kubectl expose pod events-website --type=LoadBalancer --name=events-web-svc --port=80 --target-port=8080

Install DB through Helm:
helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb

Check created DB:
helm list
helm status database-server

Create service for API and Web:
kubectl apply -f web-service.yaml 
kubectl apply -f api-service.yaml 


