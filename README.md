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
# Switch Context to Kubernetes Cluster
kubectl config use-context arn:aws:eks:us-east-1:915666190922:cluster/simple-jwt-api

# Get the list of services running
kubectl get services -o wide

# Retrieve the ELB URL
kubectl get services simple-jwt-api -o wide
```

Expected OUTPUT
```sh
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE   SELECTOR
simple-jwt-api   LoadBalancer   10.100.131.188   ac7b92c6016034116af3317f234c5dbe-419157341.us-east-1.elb.amazonaws.com   80:31815/TCP   11h   app=simple-jwt-api
```

[Endpoint](http://ac7b92c6016034116af3317f234c5dbe-419157341.us-east-1.elb.amazonaws.com) `http://ac7b92c6016034116af3317f234c5dbe-419157341.us-east-1.elb.amazonaws.com` 

### TEST API
1. Health Check:
```bash
curl http://ac7b92c6016034116af3317f234c5dbe-419157341.us-east-1.elb.amazonaws.com
```
- Expected OUTPUT
```bash
"Healthy"
```
2. Authentication:
```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"email": "test@example.com", "password": "testpassword"}' \
http://ac7b92c6016034116af3317f234c5dbe-419157341.us-east-1.elb.amazonaws.com/auth
```
- Expected OUTPUT
```bash
{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE3MzY2NDgwOTAsIm5iZiI6MTczNTQzODQ5MCwiZW1haWwiOiJ0ZXN0QGV4YW1wbGUuY29tIn0.Y-EtvTsybjtXkI6Z5DFhlgG1MBu3p8nfOUHjWKu5pQU"}
```
3. Get Contents:
```bash
curl -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE3MzY2NDgwOTAsIm5iZiI6MTczNTQzODQ5MCwiZW1haWwiOiJ0ZXN0QGV4YW1wbGUuY29tIn0.Y-EtvTsybjtXkI6Z5DFhlgG1MBu3p8nfOUHjWKu5pQU" \
http://ac7b92c6016034116af3317f234c5dbe-419157341.us-east-1.elb.amazonaws.com/contents
```
- Expected OUTPUT
```bash
{"email":"test@example.com","exp":1736648090,"nbf":1735438490}
```