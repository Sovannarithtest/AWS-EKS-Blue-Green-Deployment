To build an NGINX web server with blue-green deployment in AWS EKS (Elastic Kubernetes Service), we’ll break down the process into manageable steps. Here's a high-level guide:
Step 1: Set up AWS EKS Cluster

Ensure that you have an EKS cluster running and configured.

    Create an EKS Cluster:
        You can use AWS Management Console, AWS CLI, or Terraform to create an EKS cluster.
        Ensure you have worker nodes (EC2 instances) running in your EKS cluster.

    Sample commands to create an EKS cluster using AWS CLI:

    bash

aws eks create-cluster \
  --name blue-green-cluster \
  --role-arn arn:aws:iam::your-account-id:role/your-eks-role \
  --resources-vpc-config subnetIds=subnet-xxxx,securityGroupIds=sg-xxxx

Configure kubectl to interact with your cluster:

bash

    aws eks update-kubeconfig --name blue-green-cluster

Step 2: Deploy NGINX on EKS

Next, you need to deploy NGINX to your EKS cluster. We will create deployments for both the blue and green versions.

    Create Deployment for Blue Version: You will create a Kubernetes deployment with the blue version of your app.

    blue-deployment.yaml:

    yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-blue
  labels:
    app: nginx
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: blue
  template:
    metadata:
      labels:
        app: nginx
        version: blue
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80

Deploy it to your EKS cluster:

bash

kubectl apply -f blue-deployment.yaml

Create Deployment for Green Version: Similarly, create a deployment for the green version.

green-deployment.yaml:

yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-green
  labels:
    app: nginx
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: green
  template:
    metadata:
      labels:
        app: nginx
        version: green
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80

Deploy it:

bash

    kubectl apply -f green-deployment.yaml

Step 3: Configure Kubernetes Services for Traffic Management

You will create a service that will handle routing traffic between the blue and green deployments.

    Create Service for NGINX: This service will point to either the blue or green deployment based on the label selector.

    nginx-service.yaml:

    yaml

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
    version: blue  # initially point to the blue version
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer

Apply the service:

bash

    kubectl apply -f nginx-service.yaml

    This will create a LoadBalancer in AWS, and you can use the EXTERNAL-IP of this service to access the NGINX server.

Step 4: Blue-Green Deployment Workflow

The blue-green deployment allows you to switch traffic between the blue and green deployments smoothly.

    Initial State: Your service points to the blue deployment (version: blue). The users are directed to the blue deployment by the service.

    Update the Service for Green: To switch traffic to the green version, update the nginx-service.yaml file and change the selector to version: green:

    nginx-service-green.yaml:

    yaml

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
    version: green  # point to the green version
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer

Apply the update:

bash

    kubectl apply -f nginx-service-green.yaml

    This switches all traffic to the green deployment.

    Testing: Once you switch traffic, ensure everything is working properly with the green version by accessing the LoadBalancer IP. If everything is fine, you can terminate or scale down the blue deployment.

    You can switch back to the blue version in case of issues by changing the selector back to version: blue.

Step 5: Automating Blue-Green Deployment with Helm (Optional)

To simplify the process, you can use Helm to package the blue and green deployments and automate the switch.
Step 6: Rollback (if required)

If something goes wrong with the green version, you can quickly roll back by pointing the service back to the blue version by updating the service selector.
