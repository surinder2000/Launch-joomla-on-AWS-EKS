# Launch Joomla application on AWS EKS (Amazon Elastic Kubernetes Service) and use EFS (Elastic File System) for storage service
Launch Joomla with MySQL database on top of AWS EKS cluster

## Pre-requisites for launching application
* AWS account
* AWS CLI configured
* eksctl configured
* kubectl configured

### Create EKS cluster 

There are many ways to create an EKS cluster but I am going to create EKS cluster using yaml file. The following code create EKS cluster with three node groups

    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
        name: mycluster1
        region: ap-south-1

    nodeGroups:
     -  name: ng1
        desiredCapacity: 2
        instanceType: t2.micro
        ssh:
            publicKeyName: mykey1
     -  name: ng2
        desiredCapacity: 1
        instanceType: t2.micro 
        ssh:
            publicKeyName: mykey1 

     -  name: ng-mixed
        minSize: 2
        maxSize: 5
        instancesDistribution:
            maxPrice: 0.017
            instanceTypes: ["t2.micro", "t3.micro"]
            onDemandBaseCapacity: 0
            onDemandPercentageAboveBaseCapacity: 50
            spotInstancePools: 2
        ssh:
            publicKeyName: mykey1 

After this, to launch the cluster use the following command

    eksctl create cluster -f clusterinstancedist.yml
    
It will take around 10-20 min to create a cluster
Note: Create node groups according to your requirement

### Configure kube config file for EKS cluster
To configure kube config file for EKS cluster use the following command

    aws eks update-kubeconfig --name mycluster1
    
It is a good practice to create different namespace. So use the following command to create namespace

    kubectl create ns myns (Here myns is the name of namespace)
    
To make the created namespace default use the  following command

    kubectl config set-context --current --namespace=myns
    
### Configure EFS for getting the persistent storage for application
There are many ways to configure EFS but i am configuring from AWS console
* Go to AWS console
* Search for EFS
* Click on Create file system
* Select the VPC where cluster is created
* Select the cluster shared node security group that is configured by EKS cluster
* Click on next
* Give any name
* Leave everything as it is and go next, next and create file system
I will take a minute to create a file system

Note: Sometime it might be required to install amazon-efs-utils in each node in my case it is working without installing. To install amazon-efs-utils go to each node and run the following command

    yum install amazon-efs-utils -y
    

### Set permissions for accessing EFS by Kubernetes
Using RBAC (Role Based Access Control) authorization mechanism for Kubernetes resources we can set the permissions

To set the permissions for EFS put the following code in yaml file

    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: nfs-provisioner-role-binding
    subjects:
      - kind: ServiceAccount
        name: default
        namespace: myns
    roleRef:
      kind: ClusterRole
      name: cluster-admin
      apiGroup: rbac.authorization.k8s.io
 
     
 ### Create EFS provisioner deployment
 Put the following code in yaml file for creating EFS provisioner deployment
 
     kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: efs-provisioner
    spec:
      selector:
        matchLabels:
          app: efs-provisioner
      replicas: 1
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: efs-provisioner
        spec:
          containers:
            - name: efs-provisioner
              image: quay.io/external_storage/efs-provisioner:v0.1.0
              env:
                - name: FILE_SYSTEM_ID
                  value: fs-8865ef59
                - name: AWS_REGION
                  value: ap-south-1
                - name: PROVISIONER_NAME
                  value: mypersonal/aws-efs
              volumeMounts:
                - name: pv-volume
                  mountPath: /persistentvolumes
          volumes:
            - name: pv-volume
              nfs:
                server: fs-8865ef59.efs.ap-south-1.amazonaws.com 
                path: /
                


### Create EFS storage class and PVC for MySQL database and Joomla application
Put the following code in yaml file for creating EFS storage class and PVC for MySQL database and Joomla application

    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: aws-efs
    provisioner: mypersonal/aws-efs
    ---
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: efs-joomla
      annotations:
        volume.beta.kubernetes.io/storage-class: "aws-efs"
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 10Gi
    ---
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: efs-mysql
      annotations:
        volume.beta.kubernetes.io/storage-class: "aws-efs"
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 10Gi
          
     
 ### Create MySQL database deployment
 Put the following code in yaml file for creating MySQL database deployment
 
     apiVersion: v1
    kind: Service
    metadata:
      name: joomla-mysql
      labels:
        app: joomla
    spec:
      ports:
        - port: 3306
      selector:
        app: joomla
        tier: mysql
      clusterIP: None

    ---

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: joomla-mysql
      labels:
        app: joomla
    spec:
      selector:
        matchLabels:
          app: joomla
          tier: mysql
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: joomla
            tier: mysql
        spec:
          containers:
          - image: mysql:5.6
            name: mysql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysqlpass
                  key: password 
            - name: MYSQL_DATABASE
              value: joomla 
            ports:
            - containerPort: 3306
              name: mysql
            volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
          volumes:
          - name: mysql-persistent-storage
            persistentVolumeClaim:
              claimName: efs-mysql 
             
     
### Create Joomla application deployment 
Put the following code in yaml file for creating Joomla application deployment 

    apiVersion: v1
    kind: Service
    metadata:
      name: joomla
      labels:
        app: joomla
    spec:
      ports:
        - port: 80
      selector:
        app: joomla
        tier: frontend
      type: LoadBalancer

    ---

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: joomla
      labels:
        app: joomla
    spec:
      selector:
        matchLabels:
          app: joomla
          tier: frontend
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: joomla
            tier: frontend
        spec:
          containers:
          - image: joomla
            name: joomla 
            env:
            - name: JOOMLA_DB_HOST
              value: joomla-mysql
            - name: JOOMLA_DB_NAME
              value: joomla
            - name: JOOMLA_DB_PASSWORD 
              valueFrom:
                secretKeyRef:
                  name: mysqlpass
                  key: password 
            ports:
            - containerPort: 80
              name: joomla
            volumeMounts:
            - name: joomla-persistent-storage
              mountPath: /var/www/html
          volumes:
          - name: joomla-persistent-storage
            persistentVolumeClaim:
              claimName: efs-joomla

  
 ### Create a Kustomization file for launching Joomla application  
Put the following code in yaml file for creating a Kustomization file for launching Joomla application 

    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    secretGenerator:
         -  name: mysqlpass
            literals:
             -  password=redhat

    resources:
     -  create-rbac.yaml
     -  create-efs-provisioner.yaml
     -  create-storage.yaml
     -  mysql_deployment.yml
     -  joomla_deployment.yml


By running the kustomization file complete application will get lauched. I will set permissions for EFS, create EFS provisioner, create storage class, create pvc for MySQL and Joomla, Create MySQL deployment and mount PVC at the specified path and create Joomla deployment, mount PVC at the specified path and expose it to outside by using ELB (Elastic Load Balancer) service

To launch the applicaton run the kustomization file, go to the directory where kustomization file is saved and run the following command

    kubectl create -k .
    
![Joomla](https://github.com/surinder2000/Launch-joomla-on-AWS-EKS/blob/master/joomla.png)
    
To destroy the complete infrastructure use the following command
    
    kubectl create -k .
    
To delete the cluster use the following command

    eksctl delete cluster -f clusterinstancedist.yaml
    
    
### Thank you
   
