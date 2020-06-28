# Sugarizer School Portal

[Sugarizer](https://github.com/llaske/sugarizer) is the open-source learning platform based on Sugar that began in the famous One Laptop Per Child project.

[Sugarizer Server](https://github.com/llaske/sugarizer-server) allows the deployment of Sugarizer on a local server, for example on a school server, so expose locally Sugarizer as a Web Application. Sugarizer Server can also be used to provide collaboration features for Sugarizer Application on the network. Sugarizer Server could be deployed in a Docker container or on any computer with Node.js 6+ and MongoDB 2.6+.

Sugarizer School Portal is a new tool in the Sugarizer family which provides a way for schools interested by Sugarizer to host and manage themselves their Sugarizer deployment. It provides an on-demand (SaaS) Sugarizer Server deployment tool so that every school will be able to create a Sugarizer Server to host its own deployment without any technical skill in just a few clicks.

Under the hood, Sugarizer School Portal is a Kubernetes cluster that is able to create/manage on-demand new Sugarizer Server instances.

## Install using Helm
[Sugarizer School Portal Chart](https://github.com/nikhilm98/sugarizer-school-portal-chart/) can be used for setting up Sugarizer School Portal Server deployment on a Kubernetes cluster (Currently supports only GKE). Refer to [this](https://github.com/nikhilm98/sugarizer-school-portal-chart/) repository for setup instructions.

## Environment Setup
If you do not have a Kubernetes set-up, you can follow these steps to set-up a working development environment.
### Install MicroK8s (For Local Kubernetes set-up)
If you do not have access to [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) or any other Kubernetes provider, you can set-up a local Kubernetes environment in your system. You can follow MicroK8s [documentation](https://microk8s.io/docs/) to install MicroK8s in your system.
MicroK8s can be installed by running these commands:
```bash
# Install MicroK8s
sudo snap install microk8s --classic --channel=1.18/stable
# Join the Group
# MicroK8s creates a group to enable seamless usage of commands which require admin privilege.
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
# You will also need to re-enter the session for the group update to take place.
su - $USER
# Check if MicroK8s is up and running (Optional)
microk8s status --wait-ready
# Enable add-ons
microk8s enable dns helm3 storage
# Create Aliases (Optional)
alias kubectl='microk8s kubectl'
alias helm='microk8s helm3'
# Add bitnami helm chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```
For load balancing, you need to enable [metallb](https://metallb.universe.tf/) add-on which is an implementation of network load-balancers for bare metal clusters.
Before enabling metallb you need to find the Internal IP of your node. It can be obtained by running:
```
kubectl get nodes -o wide
```
Then to enable metallb, run:
```
microk8s enable metallb
```
Metallb will ask for an IP address range. The IPs should be on the same network as the node Internal IP. If the Internal IP of your node is `10.55.2.114` then the IP address range can be set to `10.55.2.220-10.55.2.250`.

### Install MongoDB
You can follow MongoDB [documentation](https://docs.mongodb.com/manual/installation/) to install MongoDB in your system.

### Install MongoDB Replicaset on the Kubernetes cluster
You can install MongoDB-Replicaset using [MongoDB-Replicaset](https://github.com/helm/charts/tree/master/stable/mongodb-replicaset) Helm Chart.  
You should use the `mongo-chart/values.yaml` file from [sugarizer-chart](https://github.com/NikhilM98/sugarizer-chart) repository as the values file.  
MongoDB-Replicaset can be installed by following these commands:
```bash
# Add Chart Repository
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update

# Install the chart with the release name mymongodb (You can change the release name)
# mongo-chart/values.yaml is the location of the YAML file containing values.
# You should use the `mongo-chart/values.yaml` file from `sugarizer-chart` repository as the values file.
helm install mymongodb -f mongo-chart/values.yaml stable/mongodb-replicaset
```

## Running Sugarizer School Portal
```
git clone https://github.com/NikhilM98/sugarizer-school-portal-server.git
git clone https://github.com/NikhilM98/sugarizer-chart
cd sugarizer-school-portal-server
npm install
```
You need to make some changes in the [configuration](env/config.ini) file before starting the Sugarizer School Portal. Update the [system] section in [env/config.ini](env/config.ini) according to your system.

After making the required changes. You can start Sugarizer School Portal by running this command:
```
npm start
```

To login to the Dashboard the first time, you will have to create an admin account using this command:
```
sh add-admin.sh admin password http://127.0.0.1:8080/auth/signup
```

## Server Configuration

Sugarizer School Portal configuration is load by default from file [env/config.ini](env/config.ini). Following is the typical content of the configuration file:
```
[information]
name = Sugarizer School Portal
description = Web Interface for Sugarizer School Portal

[web]
port = 8080

[security]
min_password_size = 4
max_age = 172800000
https = false
certificate_file = ../server.crt
key_file = ../server.key
strict_ssl = false
salt_rounds = 10

[database]
server = 127.0.0.1
port = 27017
name = schoolportal
waitdb = 1

[collections]
users = users
deployments = deployments

[system]
helm_binary = microk8s.helm3
kubectl_binary = microk8s.kubectl
chart_path = ../sugarizer-chart/microk8s-enviornment/sugarizer-chart/
provider = microk8s # Options: microk8s, gke
replicaset = true
databaseUrl = mymongodb

[log]
level = 1
```

The **[information]** section is for describing your Sugarizer School Portal. It could be useful for clients connected to the portal.

The **[web]** section describes the settings of the node.js process. By default, the webserver is on the port 8080.

The **[security]** section regroup security settings. `min_password_size` is the minimum number of characters for the password. `max_age` is the expiration time in milliseconds of a session with the client. At the expiration of the session, the client should reenter its password. The default time is 172800000 (48 hours). Parameters `https`, `certificate_file`, `key_file` and `strict_ssl` are explained below.
change this value if you want to use another port.

The **[database]** and **[collections]** sections are for MongoDB settings. You could update the server name (by default MongoDB run locally) and the server port. Names of the database and collections had no reason to be changed. The `waitdb` parameter allows you to force the server to wait for the database.

The **[system]** section indicates the system configuration:
- The `helm_binary` value determines the location of the helm binary file in the system. If you're using Microk8s, then setting `helm_binary` to `microk8s.helm3` will be fine.  
- The `kubectl_binary` value determines the location of the kubectl binary file in the system. If you're using Microk8s, then setting `kubectl_binary` to `microk8s.kubectl` will be fine.  
- The `chart_path` value determines the location of the sugarizer-chart folder in the system.  
- The `provider` value determines the Kubernetes cluster provider. It can be `microk8s` or `gke`.  
- The `replicaset` value determines if MongoDB Replicaset is installed in the cluster. It can be set to `true` or `false`.  
- The `databaseUrl` value determines the URL of the MongoDB database in the cluster. If `replicaset` is `true` then it is the name of the MongoDB Replicaset chart, like `mymongodb`. If `replicaset` is `false` then it is the local URL of the MongoDB database in the cluster, like `sugarizer-service-db-mymongodb.sugarizer-mymongodb.svc.cluster.local`.  

The **[log]** section indicates how the server log access. If `level` value is greater than 0 or is not present, Sugarizer School Portal will log all access to the server on the command line.

## Running Sugarizer School Portal securely using SSL (Optional)

Sugarizer School Portal could be run securely using SSL.
Few parameters in the **[security]** section of the configuration file are dedicated to that.

* To run the server securely set `https` parameter to `true`.
* `certificate_file` and `key_file` are the path to certificate and key file to sign requests.
* `strict_ssl` should be set to `false` if your certificate is a self-signed certificate or is a certificate not signed by a trusted authority.


## License

This project is licensed under `Apache v2` License. See [LICENSE](LICENSE) for full license text.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
