# EKS with Fargate workshop

This short workshop will provide you with instructions to start your journey through serverless Kubernetes in EKS. Remember you can reach me through [Twitter](https://twitter.com/ciberado), [LinkedIn](https://linkedin.com/in/javier-more) or mail (`email at javier-moreno dot com`) to solve any doubt.

## Requirements

* Install AWS command-line interface tool

```bash
pip3 install --upgrade --user awscli
aws --version
```

* To avoid an `eksctl` bug, manually configure the `aws` CLI

```
aws configure
```

* Install `eksctl`

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Cluster creation

* Configure the name of your cluster

```bash
export CLUSTER_NAME=$(whoami)cluster && echo $CLUSTER_NAME
```

* Create your cluster. It will take around 16 minutes

```bash
time eksctl create cluster \
  --name $CLUSTER_NAME \
  --region eu-west-1 \
  --fargate
```

* It is always possible to reconfigure your `.kube/config` with

```bash
aws eks --region eu-west-1 update-kubeconfig --name $CLUSTER_NAME
```

* Check how you have a node for each deployed pod:

```bash
kubectl get nodes 
kubectl get pods --all-namespaces -owide
```

## Deploy an application

* Create the deployment descriptor

```bash
ID=$(whoami)

cat << EOF > pokemon-deployment-$ID.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pokemon-deployment-$ID
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pokemonweb-$ID
  template:
    metadata:
      labels:
        app: pokemonweb-$ID
    spec:
      containers:
      - image: ciberado/pokemon-nodejs:0.0.1
        name: server
        imagePullPolicy: Always
        env:
          - name: BASE_URL
            value: $ID
EOF
```

* Apply it, and check for the result

```bash
kubectl apply -f pokemon-deployment-$ID.yaml
kubectl scale deployment pokemon-deployment-$ID --replicas 3
watch kubectl get pods --selector app=pokemonweb-$ID
```

* Read the log of any pod:

```bash
POD=$(kubectl get pods  -ojsonpath='{.items[0].metadata.name}' --selector app=pokemonweb-$ID) && echo $POD
kubectl logs $POD
```

* Run a `sh` session on it with

```bash
kubectl exec -it $POD -- /bin/sh
```

## Service creation

* Configure the service associated to the deployment (**note how ALB can be configured at this level** by using annotations)

```bash
cat << EOF > pokemon-service-$ID.yaml
apiVersion: v1
kind: Service
metadata:
  name: pokemon-service-$ID
  annotations:
    alb.ingress.kubernetes.io/target-type: ip 
    alb.ingress.kubernetes.io/healthcheck-path: "/$ID/health"
    alb.ingress.kubernetes.io/successCodes: "200"
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: pokemonweb-$ID
  type: ClusterIP
EOF
```

* Create the service

```bash
kubectl apply -f pokemon-service-$ID.yaml
```

## Ingress creation

* Configure your cluster to support the *ALB Ingress Controller*

```bash
eksctl utils associate-iam-oidc-provider \
    --region eu-west-1 \
    --cluster $CLUSTER_NAME \
    --approve
```

* Create an IAM policy to provide infrastructure permissions to the controller

```bash
aws iam create-policy \
    --policy-name ALBIngressControllerIAMPolicy \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json
```

* Update RBAC configuration of the cluster

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/rbac-role.yaml

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text) && echo $ACCOUNT_ID
eksctl create iamserviceaccount \
    --region eu-west-1 \
    --name alb-ingress-controller \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/ALBIngressControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```

* Install `jq` for *json* manipulation

```bash
sudo apt-get install jq -y
```

* Get the id of the cluster VPC (network) with

```bash
VPC_ID=$(eksctl get cluster -n $CLUSTER_NAME --output json | jq -r ".[0].ResourcesVpcConfig.VpcId") && echo $VPC_ID
```

* Compose controller configuration using variables:

```bash
R1="# - --cluster-name=devCluster/- --cluster-name=$CLUSTER_NAME"
R2="# - --aws-vpc-id=vpc-xxxxxx/- --aws-vpc-id=$VPC_ID"
R3="# - --aws-region=us-west-1/- --aws-region=eu-west-1"
```

* Get the controller manifest, replace the configuration placeholders and apply it:

```bash
curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/alb-ingress-controller.yaml | sed "s/$R1/g; s/$R2/g; s/$R3/g" | kubectl apply -f -
```

* Wait until the controller pod is correctly configured

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=alb-ingress-controller \
  --watch
```

* Create the Ingress resource and associate the service to the desired route

```bash
cat << EOF > main-ingress-$ID.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "main-ingress-$ID"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - http:
        paths:
          - path: /$ID/*
            backend:
              serviceName: "pokemon-service-$ID"
              servicePort: 80
EOF
```

* Apply the manifest

```bash
kubectl apply -f main-ingress-$ID.yaml
```

* Wait a few minutes to get the load balancing endpoint

```bash
watch kubectl describe ingress main-ingress-$ID
```

* Get the endpoint and open the application URL:

```bash
INGRESS_URL=$(kubectl get ingress main-ingress-$ID -o jsonpath={.status.loadBalancer.ingress[].hostname}) && echo http://$INGRESS_URL/$ID/
```

# Extraball: FaaS

Function as a Service is the approach that created the *serverless* trend. It is possible to implement a FaaS plataform in K8s, using [OpenFaas](https://www.openfaas.com/) to power it.

* Create the profiles associated to the namespaces that will contain the *OpenFaaS* pods

```bash
eksctl create fargateprofile \
    --region eu-west-1 \
    --cluster $CLUSTER_NAME \
    --namespace openfaas

eksctl create fargateprofile \
    --region eu-west-1 \
    --cluster $CLUSTER_NAME \
    --namespace openfaas-fn 
```

* Install *OpenFaaS* using [Arkade](https://github.com/alexellis/arkade) (and *Helm* behind it)

```bash
curl -SLfs https://dl.get-arkade.dev | sudo sh
sudo arkade install openfaas --load-balancer
```

* Check everything is up and running (it will take around two minutes)

```bash
kubectl rollout status -n openfaas deploy/gateway 
```

* Look at what you have installed:

```bash
kubectl get pods -n openfaas
kubectl get pods -n openfaas-fn 
```

* Stablish a secure proxy to access the *OpenFaaS* gateway

```bash
kubectl port-forward -n openfaas svc/gateway 8080:8080 &
```

* Install the *OpenFaas* cliente

```bash
curl -sSL https://cli.openfaas.com | sudo sh
```

* Get the *OpenFaaS* password you we can send orders to it:

```bash
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
```

* Login using

```bash
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

* Show what functions have been deployed

```bash
faas-cli list
```

* Use the store marketplace to deploy the `figlet` function

```bash
faas-cli store deploy figlet
```

* Check it has been correctly deployed:

```bash
faas-cli list
kubectl get pods -n openfaas-fn 
```

* Use it!

```bash
echo "Hello dears!" | faas-cli invoke figlet
```

* Clean up the house (**double check for unremoved resources, like ALBs**)

```bash
nohup eksctl delete cluster $CLUSTER_NAME --region eu-west-1 &
```
