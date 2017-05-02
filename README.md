[![Build Status](https://travis-ci.org/IBM/office-space.svg?branch=master)](https://travis-ci.org/IBM/office-space)
# office-space

This app mimicks the fictitious app idea from Michael Bolton in the movie "Office Space".

The app takes advantage of a financial program that computes interest for transactions by diverting fractions of a cent that are usually rounded off into a seperate bank account.

The app will be deployed on Kubernetes. The app uses a Java 8/Spring Boot webserver that computes the interest then takes the fraction of the pennies to a database. The frontend uses a Node.js app that shows the current account balance accumulated by the Spring Boot app. The backend uses a MySQL database to store the account balance.

## Prerequisite

Create a Kubernetes cluster with either [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) for local testing, or with [IBM Bluemix Container Service](https://github.com/IBM/container-journey-template) to deploy in cloud. The code here is regularly tested against [Kubernetes Cluster from Bluemix Container Service](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov) using Travis.

## Deploy to Bluemix
If you want to deploy the Office Space app directly to Bluemix, click on 'Deploy to Bluemix' button below to create a Bluemix DevOps service toolchain and pipeline for deploying the sample, else jump to [Steps](#steps)

> You will need to create your Kubernetes cluster first and make sure it is fully deployed in your Bluemix account.

[![Create Toolchain](https://github.com/IBM/container-journey-template/blob/master/images/button.png)](https://console.ng.bluemix.net/devops/setup/deploy/?repository=https://github.com/IBM/office-space)

Please follow the [Toolchain instructions](https://github.com/IBM/container-journey-template/blob/master/Toolchain_Instructions.md) to complete your toolchain and pipeline.

## Steps
1. [Create the Database services](#1-create-the-database-services)
2. [Create the Spring Boot App](#1-create-the-spring-boot-app)
3. [Create the Frontend service](#3-create-the-frontend-service)
4. [Create the Transaction Generator service](#4-create-the-transaction-generator-service)
5. [Access Your App](#5-access-your-app)

# 1. Create the Database service
The backend consists of the MySQL database and the Spring Boot app. You will also be creating a deployment controller for each to provision their Pods.
* There are two ways to create the **MySQL database** backend *(you only need to do one of them)*:
	* **A. Use Bluemix MySQL**
    Provision Compose for MySQL in Bluemix via https://console.ng.bluemix.net/catalog/services/compose-for-mysql
    Go to Service credentials and view your credentials. Your MySQL hostname, port, user, and password are under your credential uri and it should look like this
    ![images](images/mysqlservice.png)
	You will need to modify **both** `compute-interest-api.yaml` and `account-summary.yaml` files. You need to modify their environment variables to use your MySQL database in Bluemix:
    ```yaml
    compute-interest-api.yaml AND account-summary.yaml
    env:
        - name: OFFICESPACE_MYSQL_DB_USER
          value: '<your-username>'
        - name: OFFICESPACE_MYSQL_DB_PASSWORD
          value: '<Your-database-password>'
        - name: OFFICESPACE_MYSQL_DB_HOST
          value: '<Your-database-host>'
        - name: OFFICESPACE_MYSQL_DB_PORT
          value: '<your-port-number>'
    ```
    * **B. Use MySQL in container**
    **NOTE:** Leave the environment variables blank in the `compute-interest-api.yaml` and `account-summary.yaml`
```bash
$ kubectl create -f account-database.yaml
service "account-database" created
deployment "account-database" created
```

# 2. Create the **Spring Boot** App:
You will need to have [Maven installed on your environment](https://maven.apache.org/index.html).
If you want to modify the Spring Boot app, you will need to do it before building the Java project and the docker image.
* 1. Build the image
	```bash
    $ cd containers/compute-interest-api
    $ mvn package
    ```
    After Maven has successfully built the Java project, you will need to build the docker image using the provided dockerfile. *We are using Bluemix container registry to push images, but the images [can be pushed in Docker hub](https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/pull-and-push-images) as well.*
    ```bash
    $ docker build -t registry.ng.bluemix.net/<namespace>/compute-interest-api .
    ```
* 2. Push the image:
	> Note: This is being pushed in the Bluemix Container Registry.

	```bash
    $ docker push registry.ng.bluemix.net/<namespace>/compute-interest-api .
    ```
* 3. Modify `compute-interest-api.yaml` to use your image
	```yaml
    spec:
      containers:
      - image: registry.ng.bluemix.net/anthonyamanse/compute-interest-api ## replace this one with your image name
        imagePullPolicy: Always
    ```
* 4. Deploy
```bash
$ kubectl create -f compute-interest-api.yaml
service "compute-interest-api" created
deployment "compute-interest-api" created
```
> Note: The compute-interest-api multiplies the fraction of the pennies to x10,000 for simulation purposes. You can edit/remove the line `remainingInterest *= 10000` in `src/main/java/officespace/controller/MainController.java` then build the image again.

# 3. Create the Frontend service
The UI is a Node.js app that shows the total account balance.
**If you are using a MySQL database in Bluemix, don't forget to fill in the values of the environment variables in `account-summary.yaml` file, otherwise leave them blank. This was done in [Step 1](#1-create-the-database-services).**


* Create the **Node.js** frontend:
```bash
$ kubectl create -f account-summary.yaml
service "account-summary" created
deployment "account-summary" created
```

# 4. Create the Transaction Generator service
The transaction generator is a Python app that generates random transactions with accumulated interest.
* Create the transaction generator **Python** app:
```bash
$ kubectl create -f transaction-generator.yaml
service "transaction-generator" created
deployment "transaction-generator" created
```

# 5. Access Your App
You can access your app publicly through your Cluster IP and the NodePort. The NodePort should be **30080**.

* To find your IP:
```bash
$ kubectl get nodes
NAME             STATUS    AGE
169.47.241.213   Ready     7d
---OR---
$ bx cs workers <cluster-name>
ID                                                 Public IP        Private IP      Machine Type   State    Status   
kube-dal10-paac005a5fa6c44786b5dfb3ed8728548f-w1   169.47.241.213   10.177.155.13   free           normal   Ready  
```

* To find the NodePort of the account-summary service:
```bash
$ kubectl get svc
NAME                    CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
...
account-summary         10.10.10.74    <nodes>       80:30080/TCP                                                                 2d
...
```
* On your browser, go to `http://<your-cluster-IP>:30080`
![Account-balance](images/balance.png)

## References
* [John Zaccone](https://github.com/jzaccone) - The original author of the [office space app deployed via Dockers](https://github.com/jzaccone/office-space-dockercon2017).
* The Office Space app is based on the 1999 film that used that concept.

## License
[Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0)