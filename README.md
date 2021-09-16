## Setup K8s on Aws

1. Create k8s-controll Instance with image ubuntu

2. Create Role

-   Go to iam
-   Click role
-   CLick add role
-   give tag for youre role (optional)
-   add premission (ec2 full, route53 full, s3 full, iam full, vps full)

3. attach role to instance

-   go to ec2 dashbord
-   select youre instance
-   click action
-   instance seeting
-   Modify IAM role
-   select youre role
-   click save

4. Create route 53

-   go to route 53
-   click hostedzone
-   click hosted zone
-   input Domain name
-   select youre instance region
-   select youre instance vpcid
-   click created hosted zone

5. ssh to instance and install aws cli

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
apt install -y unzip python
unzip awscliv2.zip
sudo ./aws/install
```

6. install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

```

7. install kops

```bash
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops

# update kops
sudo rm -rf /usr/local/bin/kops
# and to above step
```

8. config aws cli

```bash
aws configure
```

9. create s3 bucket

```bash

aws s3api create-bucket \
    --bucket k8s-sample-store \
    --region us-east-1

# for reqion other than use-east1 use command bellow

aws s3api create-bucket \
    --bucket k8s-example-store \
    --region us-west-1 \
    --create-bucket-configuration LocationConstraint=us-west-1

# or

aws s3 mb s3://k8s-example-store


# Note: We STRONGLY recommend versioning your S3 bucket in case you ever need to revert or recover a previous state store.

aws s3api put-bucket-versioning --bucket k8s-example-store --versioning-configuration Status=Enabled

# Delete bucket
aws s3 rm s3://k8s-example-store --recursive # empty buccket before delete
aws s3api delete-bucket --bucket k8s-example-store --region us-west-1
```

10. add to .bashrc

```bash
export NAME=sunsummit.net
export KOPS_STATE_STORE=s3://k8s-sample-store
```

11. generate ssh no password

```bash
ssh-keygen

# then type
kops create secret --name $NAME sshpublickey admin -i ~/.ssh/id_rsa.pub
```

12. list availibillity zone

```bash
aws ec2 describe-availability-zones
```

13. create cluster with kops

```bash
# kops create cluster --cloud=aws --zones=us-east-1a --name=k8s.devops.com --dns-zone=k8s.devops.com --dns private

kops create cluster --cloud=aws --zones=us-east-1a --name=$NAME --node-size=t2.small --master-size=t2.small --dns-zone=$NAME --dns private

```

14. edit configuration

```bash
kops edit cluster $NAME
kops edit ig --name=$NAME nodes-us-east-1a
kops edit ig --name=$NAME master-us-east-1a
```

15. finish and create cluster

```bash
kops update cluster --name $NAME --yes --admin
```

16. check cluster

```bash
kops validate cluster --wait 10m
# or
kops validate cluster

kubectl get nodes --show-labels
```

17. ssh to master node

```bash
ssh -i ~/.ssh/id_rsa ubuntu@api.$NAME
ssh -i ~/.ssh/id_rsa ubuntu@api.sunsummit.net
```

18. delete cluster

```bash
kops delete cluster --name=$NAME --state=$KOPS_STATE_STORE --yes
```

## Setup Helm,ingress & cert-manager

1. setup youre helm like write on docs https://helm.sh/docs/intro/install/

2. install nginx ingress https://hub.kubeapps.com/charts/ingress-nginx/ingress-nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx

# make sure after install nginx thers alb created in your aws
```

3. install cert-manager with helm follow the intruction here https://hub.kubeapps.com/charts/jetstack/cert-manager

4. setup ingress with cert in here https://cert-manager.io/docs/tutorials/acme/ingress/

## Setup promethous and grafana

1. add prometheus Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

2. add grafana Helm repo

```bash
helm repo add grafana https://grafana.github.io/helm-charts

```

3. Deploy Prometheus

```bash
kubectl create namespace prometheus

helm install prometheus prometheus-community/prometheus --namespace prometheus --set alertmanager.persistentVolume.storageClass="gp2" --set server.persistentVolume.storageClass="gp2"
```

4. Prometheus components deployed as expected

```bash
kubectl get all -n prometheus
```

5. kubectl port forwarding

```bash
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

6. Deploy Grafana using below command

save this to grafana.yaml

```yml
datasources:
    datasources.yaml:
        apiVersion: 1
        datasources:
            - name: Prometheus
              type: prometheus
              url: http://prometheus-server.prometheus.svc.cluster.local
              access: proxy
              isDefault: true
```

```bash
kubectl create namespace grafana
helm install grafana grafana/grafana --namespace grafana --set persistence.storageClassName="gp2" --set persistence.enabled=true --set adminPassword='abcd1234' --values ./grafana.yaml --set service.type=LoadBalancer
```

7. Check if Grafana is deployed

```bash
kubectl get all -n grafana
```

8. Get Grafana ELB URL using this command

```bash
kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

9. Access dashboard IDs

3119/6417
