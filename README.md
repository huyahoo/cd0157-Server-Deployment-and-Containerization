# Prerequisites

* Docker Desktop - Installation instructions for all OSes can be found <a href="https://docs.docker.com/install/" target="_blank">here</a>.
* Git: <a href="https://git-scm.com/downloads" target="_blank">Download and install Git</a> for your system. 
* Code editor: You can <a href="https://code.visualstudio.com/download" target="_blank">download and install VS code</a> here.
* AWS Account
* Python version between 3.7 and 3.9.

# Setup Environment
## Create .env file
Add content as below
```
JWT_SECRET='myjwtsecret'
LOG_LEVEL=DEBUG
```

## Build Docker Image
```bash
docker build -t image_name .
```

## Run Container
```bash
docker run --name myContainer --env-file=.env -p 80:8080 myimage
```

## Docker CLI
```bash
# Check the list of images
docker image ls
# Remove any image
docker image rm <image_id>
# List running containers
docker container ls
docker ps
# Stop a container
docker container stop <container_id>
# Remove a container
docker container rm <container_id>

```

## Create an EKS Cluster, IAM Role for CodeBuild
```bash
# Ensure eksctl is installed
eksctl version

# Create the EKS cluster:
eksctl create cluster --name simple-jwt-api --nodes=2 --version=1.29 --instance-types=t2.medium --region=us-east-1

# Verify the cluster:
kubectl get nodes

# Get your AWS Account ID
aws sts get-caller-identity --query Account --output text

# Create the IAM role
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'

# Attach a permissions policy
aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json
```

### Delete stack
```bash
eksctl delete cluster simple-jwt-api --region=us-east-1
```

## Authorize CodeBuild using EKS RBAC
```bash
# Fetch
kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml

# Edit
code /System/Volumes/Data/private/tmp/aws-auth-patch.yml
# Then add 
#- groups:
#  - system:masters
#  rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
#  username: build    

# Update
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

# Apply the ConfigMap to the cluster
kubectl apply -f aws-auth-patch.yml

# Verify the mapping
kubectl get configmap aws-auth -n kube-system -o yaml
```

# Deployment

## Create JWT_SECRET
```bash
aws ssm put-parameter \
    --name "JWT_SECRET" \
    --type "SecureString" \
    --value "<YourGeneratedJWTSecret>" \
    --description "JWT secret key for Flask app" \
    --region <YourAWSRegion>
```

## Verify JWT_SECRET
```bash
aws ssm get-parameter \
    --name "JWT_SECRET" \
    --with-decryption \
    --region <YourAWSRegion>
```

## Delete JWT_SECRET
```bash
aws ssm delete-parameter --name JWT_SECRET
```

## Create Stack
```bash
aws cloudformation create-stack --stack-name myFSNDstack \
    --template-body file://ci-cd-codepipeline.cfn.yml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
        ParameterKey=GitHubToken,ParameterValue=<YourGitHubToken> \
        ParameterKey=GitHubUser,ParameterValue=<YourGitHubUsername> \
        ParameterKey=GitSourceRepo,ParameterValue=cd0157-Server-Deployment-and-Containerization \
        ParameterKey=GitBranch,ParameterValue=master \
        ParameterKey=EksClusterName,ParameterValue=simple-jwt-api
```

## Build & Deploy
- Commit changes to Github
```bash
git add .
git commit -m 'your msg'
git push
```

- Verify Deployment
```bash
kubectl get pods
kubectl get services
```

## Test
```bash
# Test Endpoint
kubectl get services simple-jwt-api -o wide

# use the external IP url to test the app
export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST <EXTERNAL-IP URL>/auth  | jq -r '.token'`
curl --request GET '<EXTERNAL-IP URL>/contents' -H "Authorization: Bearer ${TOKEN}" | jq 
```