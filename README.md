## Deployment
### 1. Create an AWS Kubernetes cluster
:warning: **If you've already created a Kubernets cluster**: Skip this step and continue with step 2!

**1.1 Create key pair**

```
~ aws ec2 create-key-pair --region eu-west-2 --key-name prodcluster --key-type rsa --query "KeyMaterial" --output text > key-pair-aws.pem
~ chmod 400 key-pair-aws.pem
```

**1.2 Create EKS cluster**

```
~ eksctl create cluster --name prod-cluster --region eu-west-2 --version=1.21 --nodegroup-name standard-nodes --node-type t3.large --nodes 1 --nodes-min 1 --nodes-max 1 --with-oidc --ssh-access --ssh-public-key prodcluster
# verify you've created successfully the Kubernetes cluster
~ kubectl get nodes -o wide
```

### 2. Create a NGINX Ingress controller and an AWS network loadbalancer 
:warning: **If you've already installed NGINX and configured the routing**: Skip this step and continue with step 3!

**2.1 Create NGINX Ingress controller and an AWS network loadbalancer**

```
~ kubectl apply -f nginx-ingress-controller.yml
# verify the installation after you applied the yaml file
~ kubectl get pods -n ingress-nginx  -l app.kubernetes.io/name=ingress-nginx
```

**2.2 Configure in the AWS dashboard Route 53 the routing**

```
~ # Go to Route 53 and add a record. 
~ # Route the traffic to the AWS network load balancer
```

### 3. Create a certificate manager and cluster issuer
:warning: **If you've already installed the certificate manager and cluster issuer that handles the TLS certificates**: Skip this step and continue with step 4!

**3.1 Create a certificate manager**

```
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.1.0 --set installCRDs=true
# verify the installation
kubectl get pods --namespace cert-manager
```

**3.2 Test if the certificate mananger workd properly**

```
kubectl apply -f cert-test-resources.yml
# check the status of the newly created certificate, you may need to wait a few seconds before cert-manager processes the certificate request
kubectl describe certificate -n cert-manager-test
# clean up the test resources
kubectl delete -f cert-test-resources.yml
```

**3.3 Create a cluster issuer**

```
kubectl create ns ind-iv
kubectl apply -n ind-iv -f cert-cluster-issuer.yml
# verify the installation
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod-issuer
kubectl describe clusterissuer letsencrypt-staging-issuer
```

### 4. Install Mattermost

**4.1 Install a Mattermost operator**

```
kubectl create ns mattermost-operator
kubectl apply -n mattermost-operator -f mattermost-operator.yml
```

**4.2 Create a RDS PostgreSQL database and a S3 file storage**

:warning: **If you've already created a database and file storage**: Skip this step and continue with step 4.3!

```
~ # Go to RDS and create a PostgreSQL database
~ # Go to S3 and create a S3 file storage
```

**4.3 Modify the yaml files to your own preferences**

Modify all setting to your own preferences. You have to add the database connection string, file storage accesskey and secretkey as a base64 encoded string. Decode the strings to see the format.

`mattermost-installation.yml`:

```yml
apiVersion: installation.mattermost.com/v1beta1
kind: Mattermost
metadata:
  name: mattermost-iv                                                         # Choose the desired name
spec:
  replicas: 1                                                                 # Adjust to your requirements
  useServiceLoadBalancer: false                                                # Set to true to use AWS or Azure load balancers instead of an NGINX controller.
  ingressName: mattermost.lucasscheepers.nl                                   # Hostname used for Ingress, e.g. example.mattermost-example.com. Required when using an Ingress controller. Ignored if useServiceLoadBalancer is true.
  ingressAnnotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/acme-challenge-type: "http01"
    cert-manager.io/cluster-issuer: "letsencrypt-prod-issuer"
  version: 6.0.0
  licenseSecret: ""                                                           # Name of a Kubernetes secret that contains Mattermost license. Required only for enterprise installation.
  database:
    external:
      secret: db-credentials                                                  # Name of a Kubernetes secret that contains connection string to external database.
  fileStore:
    external:
      url: s3.amazonaws.com                                                   # External File Storage URL.
      bucket: s3-mattermost-iv                                                # File Storage bucket name to use.
      secret: file-store-credentials
  mattermostEnv:
  - name: MM_FILESETTINGS_AMAZONS3SSE
    value: "false"
  - name: MM_FILESETTINGS_AMAZONS3SSL
    value: "false"

```

`mattermost-secrets.yml`:

```yml
# External database PostgreSQL (check https://docs.mattermost.com/install/install-kubernetes.html for MySQL)

apiVersion: v1
data:
  DB_CONNECTION_STRING: cG9zdGdyZXM6Ly91c2VyOnN1cGVyX3NlY3JldF9wYXNzd29yZEBteS1kYXRhYmFzZS5jbHVzdGVyLWFiY2QudXMtZWFzdC0xLnJkcy5hbWF6b25hd3MuY29tOjU0MzIvZGF0YWJhc2VfbmFtZT9jb25uZWN0X3RpbWVvdXQ9MTA=
kind: Secret
metadata:
  name: db-credentials
type: Opaque
---
# External file storage S3

apiVersion: v1
data:
  accesskey: YWNjZXNza2V5
  secretkey: c2VjcmV0a2V5
kind: Secret
metadata:
  name: file-store-credentials
type: Opaque
```

**4.4 Install Mattermost manifest**

```
kubectl apply -n ind-iv -f mattermost-secrets.yml
kubectl apply -n ind-iv -f mattermost-installation.yml
```

**4.5 Patch the Ingress object to add the certificate**

```
# verify if the Ingress object was created successfully
kubectl get ingress -n ind-iv -o yaml

# patch the Ingress object
kubectl apply -n ind-iv -f nginx-ingress-controller-patch.yml
# verify if the mattermost-tls certificate was issued successfully, this could take a few minutes
kubectl describe certificate -n ind-iv mattermost-tls
```

**4.6 Install ClamAV**

```
kubectl apply -n ind-iv -f clamav-deployment.yml
```

**4.7 Configure GitLab SSO and install GitLab plugin and Antivirus plugin**

`GitLab SSO`:

```
~ # Go to GitLab --> Preferences --> Applications
~ # Add an application and fill in the redirect URI's. For example:
		https://mattermost.lucasscheepers.nl/login/gitlab/complete
		https://mattermost.lucasscheepers.nl/signup/gitlab/complete
~ # Create an admin account on Mattermost
~ # Go to Mattermost --> System Console --> GitLab (Authentication)
~ # Fill in the Application ID and Application Secret key you got from GitLab
~ # Fill in the GitLab site URL
```

`GitLab plugin`:

```
~ # Go to GitLab --> Preferences --> Applications
~ # Add an application and fill in the redirect URI's. For example:
		https://mattermost.lucasscheepers.nl/plugins/com.github.manland.mattermost-plugin-gitlab/oauth/complete
~ # Go to Mattermost --> Plugin marketplace --> Install GitLab and click on configure
~ # Fill in the Application ID and Application Secret Key you got from Gitlab
~ # Fill in the GitLab site URL en regenerate the two keys
```

`Antivirus plugin`:

```
~ # Go to Mattermost --> Plugin marketplace --> Install Antivirus and click on configure
~ # Fill in the host and port number (Kubernetes uses Service Discovery)
		clamd-service:3310
```

### 5. Install ChatOps bot

**5.1 Install RabbitMQ**

```
kubectl apply -f rabbitmq-operator.yml

# give it some time to start the RabbitMQ operator
kubectl apply -n ind-iv -f rabbitmq-installation.yml

# give it some time to start the Rabbit server and then add an user with administrator permissions
kubectl exec rabbit-server-0 -n ind-iv -- rabbitmqctl add_user pyhelper pyhelper
kubectl exec rabbit-server-0 -n ind-iv -- rabbitmqctl set_user_tags pyhelper administrator
kubectl exec rabbit-server-0 -n ind-iv -- rabbitmqctl set_permissions -p / pyhelper ".*" ".*" ".*"
```

**5.2 Install PyHelper service**

```
~ # Go to Mattermost --> Integrations --> Add bot
~ # Go to the PyHelper service folder in your terminal session and open the hidden .env file
~ vim .env
~ # Modify the MATTERMOST_URL, BOT_TOKEN, BOT_TEAM and BOT_CHNNL_ID
~ # Build a docker image
~ docker image build --no-cache -t "placeholder/pyhelper" .
~ # Push the docker image
~ docker push placeholder/pyhelper
~ # Go to the PyHelper service folder in your terminal session and open the deployment.yml file
~ vim deployment.yml
~ # Modify the image variable to your own Container Registry
~ # Apply the deployment file
~ kubectl apply -n ind-iv -f deployment.yml
```

**5.3 Install GitLab service**

```
~ # Go to Mattermost --> Integrations --> Add incoming webhook
~ # Go to the GitLab service folder in your terminal session and open the hidden .env file
~ vim .env
~ # Modify the MATTERMOST_URL, GITLAB_URL and GITLAB_TOKEN
~ # Build a docker image
~ docker image build --no-cache -t "placeholder/gitlab-service" .
~ # Push the docker image
~ docker push placeholder/gitlab-service
~ # Go to the GitLab service folder in your terminal session and open the deployment.yml file
~ vim deployment.yml
~ # Modify the image variable to your own Container Registry
~ # Apply the deployment file
~ kubectl apply -n ind-iv -f deployment.yml
```