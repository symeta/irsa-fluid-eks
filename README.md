# irsa-fluid-eks
The objective of this repo is to test how to install fluid on eks with authentication by irsa rather than secret token.

## 1.preparation

- pre-requisite
  - provision an ec2
  - install eksctl on the ec2
    ```sh
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv -v /tmp/eksctl /usr/local/bin
    eksctl version
    ```
  - install kubectl on the ec2
    ```sh
    aws sts get-caller-identity
    curl --silent --location -o /usr/local/bin/kubectl \
    https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.3/2024-12-12/bin/linux/amd64/kubectl
    chmod +x /usr/local/bin/kubectl
    kubectl version --client
    ```
- provision eks cluster
  ```sh
  eksctl create cluster -f f1.yaml

  #wait till the cluster been provisioned, takes around 15 minutes

  aws eks update-kubeconfig --region us-west-2 --name f1
  ```
- get OIDC provider URL of eks cluster
  ```sh
  eksctl utils associate-iam-oidc-provider --region us-west-2 --cluster f1 --approve
  ```
- create IAM policy file
  ```sh
  cat <<EOF > fluid-eks-policy.json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "eks:DescribeCluster",
          "eks:ListClusters",
          "s3:GetObject",
          "s3:ListBucket"
        ],
        "Resource": "*"
      }
    ]
  }
  EOF
  ```
- create IAM policy
  ```sh
  aws iam create-policy --policy-name FluidEksPolicy --policy-document file://fluid-eks-policy.json
  ```
- attache the IAM pilicy with k8s serviceaccount
  ```sh
  AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

  kubectl create namespace fluid-system

  eksctl create iamserviceaccount \
    --name fluid-csi-sa \
    --namespace fluid-system \
    --cluster f1 \
    --region us-west-2 \
    --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/FluidEksPolicy \
    --approve \
    --override-existing-serviceaccounts
  ```
  
## 2.fluid installation

-  install helm
   ```sh
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   chmod 700 get_helm.sh
   ./get_helm.sh
   ```
-  install fluid
   ```sh
   helm repo add fluid https://fluid-cloudnative.github.io/charts
   helm repo update
   helm upgrade --install fluid fluid/fluid -n fluid-system \
      --set csi.kubelet.kubeConfigFile="/var/lib/kubelet/kubeconfig" \
      --set csi.kubelet.certDir="/etc/kubernetes/pki" \
      --set csi.controller.serviceAccount=fluid-csi-sa \
      --set csi.node.serviceAccount=fluid-csi-sa
   ```
- check fluid has been successfully installed
  ```sh
  kubectl get pod -n=fluid-system #all pods shoud be running
  ```

## 3.fluid alluxioruntime to cache data from s3 bucket
- create dataset, alluxioruntime pod. pvc will be auto-created. 
  ```sh
  kubectl apply -f dataset.yaml -n fluid-system
  kubectl apply -f alluxioruntime.yaml -n fluid-system
  ```
- check the status of the above
  ```sh
  # Check secret status
  kubectl get secret data-set-secret -n fluid-system

  # Check Dataset status
  kubectl get dataset s3-dataset -n fluid-system
  
  # Check AlluxioRuntime status
  kubectl get alluxioruntime s3-dataset -n fluid-system
  
  # Check PVC status
  kubectl get pvc s3-dataset -n fluid-system
  ```
## 4.create an app to read data from fluid alluxioruntime data cache

- create data-reader-pod
  ```sh
  #run the pod
  kubectl apply -f data-reader-pod.yaml -n fluid-system
  
  #check the status of the pod
  kubectl describe pod data-reader-pod -n fluid-system

  kubectl logs <specific pod name> -n fluid-system
  ```
