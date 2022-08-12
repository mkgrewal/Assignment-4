         ___        ______     ____ _                 _  ___  
        / \ \      / / ___|   / ___| | ___  _   _  __| |/ _ \ 
       / _ \ \ /\ / /\___ \  | |   | |/ _ \| | | |/ _` | (_) |
      / ___ \ V  V /  ___) | | |___| | (_) | |_| | (_| |\__, |
     /_/   \_\_/\_/  |____/   \____|_|\___/ \__,_|\__,_|  /_/ 
 ----------------------------------------------------------------- 


Hi there! Welcome to AWS Cloud9!

To get started, create some files, play with the terminal,
or visit https://docs.aws.amazon.com/console/cloud9/ for our documentation.

Happy coding!




# Increase disk space of Cloud9
# https://www.eksworkshop.com/020_prerequisites/workspace/

pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError 
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes(
    Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                os.getenv('instance_id')
            ]
        }
    ]
)
volume_id = volume_info['Volumes'][0]['VolumeId']
try:
    resize = ec2.modify_volume(    
            VolumeId=volume_id,    
            Size=30
    )
    print(resize)
except ClientError as e:
    if e.response['Error']['Code'] == 'InvalidParameterValue':
        print('ERROR MESSAGE: {}'.format(e))"
if [ $? -eq 0 ]; then
    sudo reboot
fi

# Install kubectl
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl

# install jq
sudo yum -y install jq gettext bash-completion moreutils

# Update AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install bash completion 
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion

# Add kubectl alias
echo "alias k=kubectl" >> ~/.bashrc 
 . ~/.bashrc


# Set loadBalancer Controller version
echo 'export LBC_VERSION="v2.4.1"' >>  ~/.bash_profile
echo 'export LBC_CHART_VERSION="1.4.1"' >>  ~/.bash_profile
.  ~/.bash_profile

# Configure your permanent credentials and disable Cloud9 temporary credentials
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials
aws configure

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin

# Enable eksctl bash completion
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion

# Create the cluster - this steps will take a few minutes
# https://eksctl.io/usage/creating-and-managing-clusters/
eksctl create cluster -f eks_config.yaml

# Switch to CloudFormation service, examine the resources that are being created
# Update your Kube config
aws eks update-kubeconfig --name eksworkshop-eksctl --region ${AWS_REGION}

# Create a namespace called lab4
kubectl create namespace lab4
# Install nginx in lab4 namespace
kubectl create deploy nginx --image=nginx -n lab4

# Check the pods created in lab4
kubectl get all -n lab4

# Created a new user from cloud9 terminal named mkgrewal and saving the credentials 
aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json

# Script that sets the active user to the user that we created i.e mkgrewal
cat << EoF > rbacuser_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
EoF


Task 1 and 2 

kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml
cat aws-auth.yaml
kubectl apply -f aws-auth.yaml


Task 3
kubectl apply -f clo835-role.yaml
kubectl apply -f clo835-role-binding.yaml


Task 6
k exec -it nginx -c nginx -n lab4 -- /bin/sh

KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/api/v1/namespaces/default/pods/$HOSTNAME
      
Task 7

 k run awscli0 -it --image=amazon/aws-cli -n lab4 -- ec2 describe-instances
 k run awscli3 -it --image=amazon/aws-cli -n lab4 -- s3 ls  
 
 
Task 8

eksctl utils associate-iam-oidc-provider --cluster eksworkshop-eksctl --approve


aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn'


 eksctl create iamserviceaccount \
    --name manp1 \
    --namespace lab4 \
    --cluster clo835 \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
    --approve \
    --override-existing-serviceaccounts

k get sa mkgrewal -n lab4
k describe sa -n lab4
k apply -f job-s3.yaml -n lab4  
k get job -l app=test-s3 -n lab4    
kubectl logs -l app=test-s3 -n lab4                                                                                                        
