
***create cluster with nodes

```eksctl 
  eksctl create cluster \
  --name clusterwithnode \
  --region eu-west-1 \
  --version 1.37 \
  --zones eu-west-1a,eu-west-1b \
  --nodegroup-name worker-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --node-volume-size 30 \
  --node-volume-type gp3 \
  --managed \
  --asg-access \
  --tags environment=dev,team=devops
```

eksctl utils associate-iam-oidc-provider \
  --region eu-west-1 \
  --cluster clusterwithnode \
  --approve

aws eks update-kubeconfig \
  --region eu-west-1 \
  --name clusterwithnode \
  --alias clusterwithnode

<!-- Update minimum nodes -->
eksctl scale nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes \
  --nodes-min 2

eksctl scale nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes \
  --nodes-max 5
<!-- Increase desired node -->
eksctl scale nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes \
  --nodes 4

<!-- get node -->
eksctl get nodegroup \
  --cluster clusterwithnode \
  --region eu-west-1 \
  --name worker-nodes





<!-- Create cluster without nodegroup -->
  eksctl create cluster \
  --name clusterwitouthnode \
  --version 1.37 \
  --region eu-west-1 \
  --without-nodegroup

  eksctl utils associate-iam-oidc-provider \
  --region eu-west-1 \
  --cluster clusterwitouthnode \
  --approve

 eksctl create nodegroup \
  --cluster clusterwitouthnode \
  --region eu-west-1 \
  --name without-worker-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --node-volume-size 30 \
  --node-volume-type gp3 \ 
  --managed


aws eks update-kubeconfig \
  --region eu-west-1 \
  --name clusterwitouthnode \
  --alias clusterwitouthnode

<!-- helm charts for gateway class -->
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.10.0 -n envoy-gateway-system --create-namespace


kubectl wait --timeout=5m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available

  <!--  -->

  aws cloudformation update-termination-protection --region eu-west-1 --stack-name eksctl-withnodecluster-cluster --no-enable-termination-protection

  aws cloudformation delete-stack \
  --region eu-west-1 \
  --stack-name eksctl-withnodecluster-cluster


  aws cloudformation wait stack-delete-complete \
  --region eu-west-1 \
  --stack-name eksctl-withnodecluster-cluster


  <!-- Add ssl -->

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true

kubectl get pods -n cert-manager