# Install Kubectl & EKSCTL

# To create default cluster
eksctl create cluster 

# To create customized eks cluster - folloe below steps
# step:1 - create cluster from manifest file
eksctl create cluster -f manifest_files/1-cluster-config.yaml

# step:2 - need to install python as we will be using python to create IAM role policy

cd manifest_files
python3 -m venv python/venv
source python/venv/bin/activate
pip install -r python/requirements.txt


# step:3 -Creating required IAM role and policy for AWS Load Balancer components
python3 ./python/create_eks_iam_role.py
kubectl apply -f ./2-aws-load-balancer-controller-service-account.yaml

# step 4 - Deploying cert-manager and then waiting for cert-manager deployments to finish
kubectl apply -f ./3-cert-manager.yaml
kubectl wait --for=condition=available --timeout=120s deployment/cert-manager -n cert-manager
kubectl wait --for=condition=available --timeout=120s deployment/cert-manager-cainjector -n cert-manager
kubectl wait --for=condition=available --timeout=120s deployment/cert-manager-webhook -n cert-manager

# step:5 - Update manifest files containing our custom VPC ID information
python3 ./python/update_manifests.py

# step:6 Deploying load balancer controller configs and waiting for the ingress controller config to finish creating
kubectl apply -f ./4-load-balancer-controller-config.yaml
kubectl wait --for=condition=available --timeout=120s deployment/aws-load-balancer-controller -n kube-system

# step:7 deploy sample application
kubectl apply -f ./5-deployment-config.yaml
kubectl wait --for=condition=available --timeout=120s deployment/deployment-2048 -n game-2048

# step:8 deploying service for application
kubectl apply -f ./6-service-config.yaml

# step:9 Deploying our Ingress Config and our Ingress (Please give the ALB a few minutes to deploy before testing)
kubectl apply -f ./7-ingressClass-config.yaml
kubectl apply -f ./8-ingress-config.yaml --validate=false

# step:10 Checking Ingress status
kubectl get ingress/ingress-2048 -n game-2048

# step: 11 Deploying TLS Version
## Remember to first create your ACM TLS Cert, as well as the required Route 53 records for your ALB. Also, remember to remove the dualstack. from the Alias record name!