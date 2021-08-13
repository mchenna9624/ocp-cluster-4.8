# Installing a cluster on user-provisioned infrastructure in AWS by using CloudFormation templates #

Firstly I would like to say great thanks to RedHat Team for their great documentation to deploy Openshift Cluster on different cloud providers using user-provisioned infra. Please find below link which I reffered and deployed this cluster.

https://docs.openshift.com/container-platform/4.8/installing/installing_aws/installing-aws-user-infra.html#installing-aws-user-infra

While the above document describes detailed way to deploy cluster, here I am going to concentrate on highlevel stpes what I followed to deploy cluster. for details please refer official document on above link.

Cluster URL which I built following the guide : https://console-openshift-console.apps.mohit.aws-ocpcluster.click 

GitHub URL for cloud formation templates used in this guide: https://github.com/mchenna9624/ocp-cluster-4.8

Steps:

1. Create AWS Account 
2. Create a User with Administration Access and save AccessKey ID and Secret Token
3. Set up AWS Configuration in your local box using above access key and secret id, also set up default zone where you want to build the cluster.
4. Create an SSH key and add it to an agent on the installer instance
    a. $ cd ~/.ssh
    b. $ ssh-keygen -t rsa -b 4096 -N '' -f <<file name, usually id_rsa>>   
    c. $ eval "$(ssh-agent -s)"
    d. $ ssh-add rsa_key_1
5. Obtain the installation program
    a. Access infra provider (https://console.redhat.com/openshift/install) page
    b. Select AWS on Run it your self section 
    c. Select User Provisioned Infrastructure
    d. Download installer and pull secret
    e. extract the downloaded installer to some directory.
6. Create the Installation configuration file by using below command and follow the onscreen instructions. By Default it will be have single availability zone and 3 master and 3 workers. If you want to adjust them, you can tweek it here.

    ./openshift-install create install-config --dir=<installation_directory> 

7. Create kubernetes manifest and Ignition Config files.

    a. ./openshift-install create manifests --dir=<installation_directory> 
    b. rm -f <installation_directory>/openshift/99_openshift-cluster-api_master-machines-*.yaml
    c. rm -f <installation_directory>/openshift/99_openshift-cluster-api_worker-machineset-*.yaml
    d. ./openshift-install create ignition-configs --dir=<installation_directory> 

    Refer Paraeters and Template files from official document or from my GitHub and replace values accordingly.

8. Create VPC in AWS 

    a. aws cloudformation create-stack --stack-name <stack name> --template-body file://<template for vpc>.yaml --parameters file://<parameters for vpc>.json 
    
8. Creating networking and load balancing components in AWS

    a. aws cloudformation create-stack --stack-name <stack name> --template-body file://<template for network and lb>.yaml --parameters file://<parameters for network and lb>.json --capabilities CAPABILITY_NAMED_IAM

9. Creating security group and roles in AWS

    a. aws cloudformation create-stack --stack-name <stack name> --template-body file://<template for securitygroup and roles>.yaml --parameters file://<parameters for security group and roles>.json --capabilities CAPABILITY_NAMED_IAM

10. Creating the bootstrap node in AWS

    a. Create S3 Bucket: aws s3 mb s3://<cluster-name>-infra 
    
    b. Upload bootstrap ignition file to bucket: aws s3 cp <installation_directory>/bootstrap.ign s3://<cluster-name>-infra/bootstrap.ign 

    c. aws cloudformation create-stack --stack-name <stack name> --template-body file://<template for bootstrap_node>.yaml --parameters file://<parameters for bootstrap_node>.json --capabilities CAPABILITY_NAMED_IAM

11. Creating the control plane machines in AWS

    a. aws cloudformation create-stack --stack-name <stack name> --template-body file://<template for ,master_nodes>.yaml --parameters file://<parameters for master_nodes>.json

12. Creating the worker nodes in AWS (Run 3 times for 3 worker nodes with different stack names)

    a. aws cloudformation create-stack --stack-name <stack name> --template-body file://<template for ,worker_node>.yaml --parameters file://<parameters for worker_node>.json

13. Start the bootstrap sequence to initialize the Openshift Container Platform Control Plane

    a. ./openshift-install wait-for bootstrap-complete --dir=<installation_directory> --log-level=info 

14. Logging in to the cluster by using the CLI

    a. export KUBECONFIG=<installation_directory>/auth/kubeconfig 
    b. oc whoami   expected result is system:admin
    c. oc get nodes (Should display both master and worker nodes)

    if worker nodes are not displayed follow the below steps to approve all the csrs

    d. oc get csr
    c. oc adm certificate approve <name of csr which is in pending state in above command.>

    approve all csr by running above command until you see all csr are in approve state

15. Make sure all clusteroperators are available

    a. oc get clusteroperators  (it took some time for all the operators appear in available state)

16. Delete Bootstrap Node

    a. aws cloudformation delete-stack --stack-name <name> 

17. Completing an AWS installation on user-provisioned infrastructure

    a. ./openshift-install --dir=<installation_directory> wait-for install-complete 

    You should be able to login to console by using output of above command.


Next objective is adding authentication mechanism for regular users for the cluster. All the best Guys.
