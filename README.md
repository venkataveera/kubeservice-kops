# Configuring Kubernetes Cluster on AWS and running/scaling a .Net Core application

<!-- TOC -->

- [Development Environment Setup](#development-environment-setup)
- [Containerize the application](#containerize-the-application)
- [Push the image to DockerHub](#push-the-image-to-dockerhub)
- [Configure Kubernetes Cluster on AWS](#configure-kubernetes-cluster-on-aws)
- [Deploy the Kubernetes Dashboard](#deploy-the-kubernetes-dashboard)
- [Deploy the Application in Kubernetes using .yml file](#deploy-the-application-in-kubernetes-using-yml-file)
- [Scaling Kubernetes Cluster pods](#scaling-kubernetes-cluster-pods)
- [Delete the Kubernetes Cluster](#delete-the-kubernetes-cluster)
<!-- /TOC -->

### Development Environment Setup
- Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)
- Install [Kubernetes Operations (kops)](https://github.com/kubernetes/kops#installing)

### Containerize the application
- Clone this repo to a folder
- Open a command prompt and navigate to your project folder.
- Use the following commands to build and run your Docker image:
	```
	docker build . -t kubeservice:local
	docker run -d -p 8000:80 kubeservice:local
	```
- View the .Net Core application running from a container by navigating to  [localhost:8000](http://localhost:8000/)

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubeservice-local-docker.png)

### Push the image to DockerHub
- Log in on https://hub.docker.com/
- Click on Create Repository.
- Choose a name (e.g. kubeservice) and a description for your repository and click Create.
- Log into the Docker Hub from the command line
	```
	docker login --username=yourhubusername --email=youremail@company.com
	```
	just with your own user name and email that you used for the account. Enter your password when prompted. If everything worked you will get a message similar to

	```
	WARNING: login credentials saved in /home/username/.docker/config.json
	Login Succeeded
	```
- Check the image ID using
	```
	docker images
	```
	and what you will see will be similar to

	```
	REPOSITORY                 TAG                  IMAGE ID            CREATED             SIZE
	kubeservice                local                a847aeef68b9        2 hours ago        257MB
	<none>                     <none>               92e722450fdb        2 hours ago        1.74GB
	microsoft/dotnet           sdk                  6baac5bd0ea2        3 weeks ago         1.73GB
	microsoft/dotnet           aspnetcore-runtime   1fe6774e5e9e        3 weeks ago         255MB
	```
- Tag image
	```
	docker tag a847aeef68b9 yourhubusername/kubeservice:latest
	```
- Push image to the repository created in previous steps
	```
	docker push yourhubusername/kubeservice
	```
### Configure Kubernetes Cluster on AWS
- Login to your [AWS console](https://aws.amazon.com/account/)
- Generate access keys for your user by navigating to Users/Security credentials page
- Make sure your IAM user has following permissions:
	```
	AmazonEC2FullAccess
	AmazonRoute53FullAccess
	AmazonS3FullAccess
	AmazonVPCFullAccess
	IAMFullAccess
	```
- Make sure to configure the AWS CLI to use your access key ID and secret access key
	```
	$ aws configure
	AWS Access Key ID [None]: <AccessKeyValue>
	AWS Secret Access Key [None]: <SecretAccessKeyValue>
	Default region name [None]: <us-east-1>
	Default output format [json]:
	```
- Create an S3 bucket for `kops` to use to store the state of the Kubernetes cluster and its configuration
	```
	$ bucket_name=vv-kops-state-store
	$ aws s3api create-bucket --bucket ${bucket_name} --region us-east-1
	```
- Enable versioning to revert or recover a previous state store.
	```
	$ aws s3api put-bucket-versioning --bucket ${bucket_name} --versioning-configuration Status=Enabled
	```
- Set the Kubernetes cluster name and S3 bucket URL environment variables
	```
	$ export KOPS_CLUSTER_NAME=vv.k8s.local
	$ export KOPS_STATE_STORE=s3://${bucket_name}
	```
- Generate Kubernetes cluster configuration using kops
	```
	$ kops create cluster --node-count=2 --node-size=t2.medium --zones=us-east-1a --name=${KOPS_CLUSTER_NAME}
	```
- Finally, build the Kubernetes cluster on AWS using following kops command. This might take a few minutes to boot the EC2 instances and download the Kubernetes components.

	```
	$ kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
	```
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-aws-images.png)

- Validate the cluster to ensure the master + 2 nodes have launched

	```
	$ kops validate cluster
	Validating cluster vv.k8s.local

	INSTANCE GROUPS
	NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
	master-us-east-1a	Master	c4.large	1	1	us-east-1a
	nodes			Node	t2.medium	2	2	us-east-1a

	NODE STATUS
	NAME				ROLE	READY
	ip-172-20-37-8.ec2.internal	master	True
	ip-172-20-41-188.ec2.internal	node	True
	ip-172-20-43-113.ec2.internal	node	True

	Your cluster vv.k8s.local is ready
	```
- Finally, you can see your Kubernetes nodes with kubectl

	```
	$ kubectl get nodes
	NAME                            STATUS   ROLES    AGE   VERSION
	ip-172-20-37-8.ec2.internal     Ready    master   3m    v1.10.6
	ip-172-20-41-188.ec2.internal   Ready    node     2m    v1.10.6
	ip-172-20-43-113.ec2.internal   Ready    node     2m    v1.10.6
	```

### Deploy the Kubernetes Dashboard
- Deploy the Kubernetes Dashboard by running following command
	```
	$ kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

	secret/kubernetes-dashboard-certs created
	serviceaccount/kubernetes-dashboard created
	role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
	rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
	deployment.apps/kubernetes-dashboard created
	service/kubernetes-dashboard created
	```
- Access Dashboard using the kubectl command-line tool by running the following command
	```
	$ kubectl proxy

	Note: Kubectl will handle authentication with apiserver and make Dashboard available at http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
	The UI can only be accessed from the machine where the command is executed.
	```
- Execute the below command to find the admin service account token
	```
	$ kops get secrets admin --type secret -oplaintext
	```
- Provide the above service account token on the service token request page
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-dashboard-token.png)

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-dashboard.png)

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-about.png)

### Deploy the Application in Kubernetes using .yml file
- Below is the content of `kubeservice-deploy.yml` file:
	```
	apiVersion: apps/v1
	kind: Deployment
	metadata:
		name: kubeservice-deployment
		labels:
		    app: kubeservice
	spec:
		replicas: 3
		template:
			metadata:
				name: kubeservice
				labels:
					app: kubeservice
			spec:
				containers:
			      - name: kubeservice
			        image: venkataveera/kubeservice:latest
			        imagePullPolicy: IfNotPresent
			    restartPolicy: Always
		selector:
		    matchLabels:
			      app: kubeservice
	---
	apiVersion: v1
	kind: Service
	metadata:
		 name: kubeservice-service
	spec:
	  selector:
	    app: kubeservice
	  ports:
	    - port: 80
	  type: LoadBalancer

- Click on + CREATE on top right corner of the Kubernetes Dashboad
- Navigate to 'CREATE FROM FILE' tab and select the `kubeservice-deploy.yml` file

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubeservice-deployment-using-yml.png)

- Finally, click on UPLOAD button.  This will deploy the application and dashboard looks like below once the deployment is completed.

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubeservice-deployment-success.png)

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-services-dashboard.png)

- Access the application using external endpoint listed in Kubernetes - Services Dashboard

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubeservice-from-kops-instance-1.png)

![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubeservice-from-kops-instance-3.png)

### Scaling Kubernetes Cluster pods
- Verify the current replicas # and deployment of application
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubeservice-deployment-success.png)

- Navigate to Deployments and click on Scale as shown in below screen
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-deployment-scale.png)

- Change the `Desired number of pods` to 5 instead of 3
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-deployment-scaled-to-5.png)

- Click OK to scale the pods to 5
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-deployment-scale-in-progress.png)

- Finally, dashboard looks like this once the deployment is completed
![alt text](https://github.com/venkataveera/kubeservice-kops/blob/master/content/kubernetes-deployment-scale-completed.png)

### Delete the Kubernetes Cluster
- Finally, when you are ready to tear down your Kubernetes cluster, you can delete the cluster using following command

	```
	kops delete cluster --name ${KOPS_CLUSTER_NAME} --yes
	```
