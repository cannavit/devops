
# Create Infrastructure with Terraform

!> **Important** These commands can change the infrastructure. Do not use in production. These changes must be evaluated before being applied. It can cause interruptions or intermittent services.

Go to folder

    cd terraform 

#### Init Terrafrom 

    terraform init

#### Terraform PLan

    terraform plan

#### Apply Configuration and Create K8s Cluster

    terraform apply



#### Connect with the cluster using:

    aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)

#### Connect with the cluster 

    aws eks --region eu-south-1 update-kubeconfig --name fantaking-dunkest

#### Check Connection With the cluster

Before continuing make sure you have access to the cluster, you can run the following command to check that you can see the EKS cluster

    kubectl get nodes

Expected response

    NAME                                            STATUS   ROLES    AGE   VERSION
    ip-172-31-45-0.eu-south-1.compute.internal    Ready    <none>   16d   v1.21.5-eks-9017834
    ip-172-31-7-238.eu-south-1.compute.internal   Ready    <none>   56m   v1.21.5-eks-9017834

Must see one Node a cluster node