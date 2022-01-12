# TKG 1.4 with NSX-ALB (vSphere Network)

- [TKG 1.4 with NSX-ALB (vSphere Network)](#tkg-14-with-nsx-alb-vsphere-network)
  - [Important note](#important-note)
  - [NSX ALB deployment](#nsx-alb-deployment)
    - [Network Configuration Planning](#network-configuration-planning)
    - [Network Configuration](#network-configuration)
      - [Front-End Network](#front-end-network)
      - [Management K8s Network](#management-k8s-network)
      - [Management Network](#management-network)
    - [Controller Deployment](#controller-deployment)
    - [Initial Configuration](#initial-configuration)
    - [Form the cluster](#form-the-cluster)
    - [Patching](#patching)
    - [Cloud Configuration](#cloud-configuration)
    - [IPAM and DNS profiles](#ipam-and-dns-profiles)
    - [SE Configuration](#se-configuration)
    - [Certificate](#certificate)
    - [License](#license)
    - [Routing](#routing)
  - [Preparing an Ubuntu VM](#preparing-an-ubuntu-vm)
    - [Docker installation](#docker-installation)
    - [Helm Installation](#helm-installation)
    - [Kubectl, Tanzu & Helm completion](#kubectl-tanzu--helm-completion)
  - [Tanzu Kubernetes Grid Deployment](#tanzu-kubernetes-grid-deployment)
    - [Prerequisites](#prerequisites)
      - [SSH Port Forwarding](#ssh-port-forwarding)
      - [SSH Key](#ssh-key)
    - [Deploy the management cluster](#deploy-the-management-cluster)
  - [Tanzu Kubernetes Grid Post Deployment](#tanzu-kubernetes-grid-post-deployment)
    - [Authenticate as admin on the Management Cluster](#authenticate-as-admin-on-the-management-cluster)
    - [Verify that all apps where deployed successfully](#verify-that-all-apps-where-deployed-successfully)
    - [Pinniped](#pinniped)
    - [Configure DNS Delegation (NSX ALB or External-DNS)](#configure-dns-delegation-nsx-alb-or-external-dns)
      - [NSX ALB (Enterprise Edition)](#nsx-alb-enterprise-edition)
      - [External-DNS](#external-dns)
      - [Windows DNS](#windows-dns)
    - [Configure NSX ALB for Workload Clusters](#configure-nsx-alb-for-workload-clusters)
      - [L7 Ingress controller (NSX ALB Enterprise Edition)](#l7-ingress-controller-nsx-alb-enterprise-edition)
        - [L7 Ingress in ClusterIP Mode](#l7-ingress-in-clusterip-mode)
      - [L4 LB Only (NSX ALB Essentials Edition)](#l4-lb-only-nsx-alb-essentials-edition)
    - [Configure a wildcard certificate for Ingress Controller](#configure-a-wildcard-certificate-for-ingress-controller)
      - [NSX-ALB as Ingress](#nsx-alb-as-ingress)
      - [Contour as ingress](#contour-as-ingress)
  - [Deploying Guest Clusters](#deploying-guest-clusters)
    - [Create KubeConfig files for Cluster Access](#create-kubeconfig-files-for-cluster-access)

<!-- pagebreak -->
## Important note

==**Please make sure to read the official VMware documentation as well. Especially the one related to the specific version you're going to deploy.Tanzu Kubernetes Grid is a work in progress and lots of procedures change between each release**==

## NSX ALB deployment

NSX ALB (also known as AVI Networks) is a distributed load balancer that can be used for Tanzu environments.
It's going to be used to provide VIPs for Kubernetes Control Plane as well as for any application that requires a service of type "Load Balancer". It's a replacement of both HA-Proxy and MetalLB for Tanzu with vSphere network configuration.

NSX Advanced Load Balancer includes the following components:

- **Avi Controller** manages VirtualService objects and interacts with the vCenter Server infrastructure to manage the lifecycle of the service engines (SEs). It is the portal for viewing the health of VirtualServices and SEs and the associated analytics that NSX Advanced Load Balancer provides. It is also the point of control for monitoring and maintenance operations such as backup and restore.
- **Avi Kubernetes Operator (AKO)** is a Kubernetes controller that each cluster runs on one of its nodes. Each AKO pod uses its cluster's Kubernetes API to watch for changes in the cluster's LoadBalancer and Ingress specifications, or other relevant custom resource definitions. When the AKO detects a change, it calls the Avi Controller API to make the change in the Avi resources, for example create a new load balancer VirtualService object and connect it with pods running in the cluster.
- **AKO Operator** on the management cluster manages the lifecycle and configuration of the AKO on each workload cluster, and can make runtime changes to the AKO configuration.
- **Service Engines (SE)** implement the data plane in a VM.
- **SE Groups** group Service Engines into isolated sets, for example to dedicate them to specific namespaces. This lets you control SEs collectively and set maximum SE counts for different resource types, such as CPU and Memory.

### Network Configuration Planning

As a picture is worth thousand words, here's a diagram of the network configuration as we deployed in our lab :

![](./images/tkg-nsxalb.drawio.png)

The following networks are configured on our vSphere / NSX environment to host Tanzu Kubernetes Grid :

|Name|Role|Type|CIDR|DHCP|
|---|---|---|---|---|
|mgmt-k8s-ingress|Front-End Network|NSX-T Overlay|10.30.230.64/27|No|
|demo-tkg-12|Management K8s|NSX-T Overlay|10.30.231.0/24|**Yes**|
|Management Network|Management VMware|VLAN|10.30.224.0/25|No|

**It's important to enable DHCP on the Management K8s network (demo-tkg-12 here). It will allow Kubernetes nodes (both master and worker) as well as AVI Service Engines to receive an IP address.**

### Network Configuration
This section describes the configuration of each segment created for TKG deployment (including NSX ALB requirements)
#### Front-End Network
The front-end network is going to host every VIPs related to TKG.
It can either be for the control plane of Kubernetes clusters, or for applications deployed on our Kubernetes clusters.

Here, we are using mgmt-k8s-ingress as this segment / port group. 

![](images/front-end-segment.png)
#### Management K8s Network
The Management K8s network will be used as the default network for each VMs that tanzu deploys (Kubernetes Master nodes, Kubernetes Worker Nodes, AVI Service Engines (SE)). 

As VMs on this network will be provisioned using Cluster API (CAPV), it requires a DHCP with both DNS and NTP options configured.

On our lab, we are using demo-tkg-12 NSX-T segment for this purpose.
![](images/management-k8s.png)
And the DHCP configuration :
![](images/management-k8s-dhcp.png)

#### Management Network
The Management Network is used to provide management IP adresse to AVI controllers as well as AVI Service Engines (SE).

In our LAB environment, this network is the VLAN where we also have vCenter and ESX management adress, the portgroup is named Management Network-08877285-62bb-4fa2-9ff2-e63c81af33a3.

We need at least **6** IP addresses available on this network:
- 4 for AVI Controllers (3 controllers + 1 VIP)
- 2 for AVI SE (a pair of SEs is created for each Kubernetes cluster deployed)

### Controller Deployment

**Before deploying, create DNS records. It's mandatory to have a working cluster, especially when you want to upgrade it.**

1. Create the DNS records
![](images/avi-dns-records.png)
2. Download AVI Controller OVAs from VMware site (TKG download page). **Make sure to download the version specified on the TKG Documentation (version 20.1.6 or 20.1.3 for TKG 1.4)**
3. Download the latest patch corresponding to the release of NSX ALB you selected
4. Deploy **3** times the same OVA to form a cluster
  ![](images/avi-ova-deploy.png)
4. Power On each controller

### Initial Configuration

1. Using your browser, connect to one controller
2. Set the new admin password
  ![](images/avi-ctr-password.png)
3. Enter the passphrase, DNS resolver, DNS Search Domain, SMTP information
  ![](images/avi-ctr-initial.png)
4. On the Multi-Tenant part, let the default settings

**Make sure to do this for only one controller before going to the next step**
### Form the cluster
1. Connect to the controller you previously configured via its web admin interface
2. Go to Administration > Controller > Nodes and click the Edit button
  ![](images/avi-ctr-cluster-01.png)
3. Fill the form to create the cluster
  ![](images/avi-ctr-cluster-02.png)
4. Wait 5 minutes for the process of cluster creation to start

### Patching
Once the cluster is running, we can patch it with the latest version we found on the NSX ALB download site.
1. Go to **Administration > Controller > Software**. And click **Upload From Computer**
2. Select the patch you previously downloaded
3. Once the transfert is complete, go to **Administration > Controller > System Update**
4. Select the patch and click on **Upgrade**
  ![](images/avi-ctr-upgrade.png)
5. Leave all options by default and click on **Continue** and **Confirm**

### Cloud Configuration
The next step is to configure the Cloud. In our case, our vCenter and networks related to it.
The Management Network we are going to configure in this wizzard will be used as the Management interface for all AVI Service Engines (SE).
1. Go to **Infrastructure > Clouds**. Click on **Create > VMware vCenter/vSphere ESX**
2. Fill in the name (the name of the vCenter in our case)
3. Fill in the login informations and let all other options by default.
    ![](images/avi-ctr-cloud-01.png)
4. Select the Datacenter and leave all other options by default
5. Select the Management Network, the Default Service Engine Group and fill in the information for the management IP adresses that will be used for AVI SE.
  ![](images/avi-ctr-cloud-02.png)

==For each subnet to be configured on NSX ALB, use the real CIDR instead of the one you can see on NSX-T interface (which is the GW address instead of the CIDR)==

### IPAM and DNS profiles
In this step, we are going to create IPAM and DNS profiles. The IPAM profile will be used to provide VIPs addresses in the front-end network *(mgmt-k8s-ingress)*.
1. Go to **Templates > Profiles > IPAM/DNS profiles** and click on **Create > IPAM Profile**
2. Fill in the name and click on + Add Usable Network and select the front-end Network
  ![](images/avi-ctr-ipam.png)
3. Go to **Templates > Profiles > IPAM/DNS profiles** and click on **Create > DNS Profile**
4. Give a name to the template and insert the domain name you want to use
  ![](images/avi-ctr-dns.png)
5. Go back to **Infrastructure > Clouds** and click on the Cloud we previously created.
6. On the Infrastructure tab, select the IPAM and DNS profiles we just created.
  ![](images/avi-ctr-cloud-03.png)
7. Go to **Infrastructure > Networks > Select Cloud** and click on the edit button of the Front-End network (*mgmt-k8s-ingress*)
8. Click on **+Add Subnet**, and fill in the CIDR of the Front-End network (*10.30.230.64/27*)
9. Then click on **+Add Static IP Address Pool**, and fill the IP range you want for your Front-End IP addresses (*10.30.230.66-10.30.230.90*)
  ![](images/avi-ctr-ipam-02.png)
10. Go back to **Infrastructure > Networks > Select Cloud** and click on the edit button of the Management K8s network (*demo-tkg-12*)
11. Tick the "DHCP Enabled Box"
  ![](images/avi-ctr-ipam-03.png)

### SE Configuration
AVI Service Engines will be deployed automatically, through AKO. We need to configure some settings to be sure that the SE will be deployed on the correct datastore, with a name we chose.
1. Go to **Infrastructure > Service Engine Group**, select the Cloud (at the top of the page) and edit the Default-Group
2. Click on the **Advanced** Tab
3. Select the prefix you want to use, the vSphere Cluster, and the datastore
  ![](images/avi-ctr-se.png)

### Certificate
Follow VMware's documentation for the certificate. There is no trap for this one

### License
You can either have the essentials or the Enterprise license Tier.

If you have the Enterprise license, you'll be able to use those features :
- DNS Delegation
- L7 Ingress through NSX ALB
- GSLB
- ...

Follow VMware's documentation for License. There is no trap for this one

### Routing
1. Go to **Infrastructure > Routing > Select Cloud : *Your Cloud*** and click **Create**
2. Insert 0.0.0.0/0 as the **Gateway subnet** and insert the gateway of the Front-End subnet as the **Next Hop**
  ![](images/avi-ctr-routing.png)

<!-- pagebreak -->
## Preparing an Ubuntu VM

### Docker installation

1. First we need to update our repositories and install prerequisites
    ```bash
    sudo apt-get update
    sudo apt-get install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release
    ```
2. Add Docker’s official GPG key:
    ```bash
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```
3. Use the following command to set up the stable repository
    ```bash
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
4. Update the apt package index, and install the latest version of Docker Engine and containerd
    ```bash
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io
    ```
5. Add your user to the docker group
    ```bash
    sudo usermod -aG docker $USER
    ```
6. Log out and log back in so that your group membership is re-evaluated
7. Verify that you can run docker commands **without** sudo
    ```bash
    docker run hello-world
    ```

### Helm Installation

1. Install helm following the official documentation : https://helm.sh/docs/intro/install/


### Kubectl, Tanzu & Helm completion
To allow autocompletion of kubectl, tanzu  and helm commands, run the following commands (be sure to have executed kubectl and tanzu commands at least once before) :
```bash
vmware@tkg-jump:~$ sudo apt install bash-completion
```
```bash
vmware@tkg-jump:~$ kubectl completion bash > ~/.kube/completion.bash.inc
  printf "
  # Kubectl shell completion
  source '$HOME/.kube/completion.bash.inc'
  " >> $HOME/.bash_profile
  source $HOME/.bash_profile
```
```bash
vmware@tkg-jump:~$ tanzu completion bash > ~/.kube-tkg/completion.bash.inc
  printf "
  # Tanzu shell completion
  source '$HOME/.kube-tkg/completion.bash.inc'
  " >> $HOME/.bash_profile
  source $HOME/.bash_profile
```

```bash
vmware@tkg-jump:~ mkdir ~/.helm
vmware@tkg-jump:~$ helm completion bash > ~/.helm/completion.bash.inc
  printf "
  # Helm shell completion
  source '$HOME/.helm/completion.bash.inc'
  " >> $HOME/.bash_profile
  source $HOME/.bash_profile
```

<!-- pagebreak -->
## Tanzu Kubernetes Grid Deployment

To deploy your first TKG management cluster, you have to have :
- NSX ALB deployed and configured
- A bootstrap VM with all the required tools installed (docker, kubectl...). We use an Ubuntu VM in our scenario

First we are going to use the UI installer, as it will help us fill the YAML file to configure the management cluster.
Before finishing and running the installation with the UI, we are going to use the command line provided to effectively launch the deployment. It will allow us to have a better verbosity on what happens on the background.

### Prerequisites

#### SSH Port Forwarding
If you run the UI installer on the VM or a Linux that doesn't have a browser, it can be helpful to redirect the port 8080 of the bootstrap VM to your machine. For this, execute the following command on **your machine**.
```bash
ssh -L 8080:localhost:8080 vmware@10.30.228.17
```

#### SSH Key

To create an SSH Key pair that is going to be used for tanzu, follow these steps on a Linux machine :
```bash
ssh-keygen -t ed25519 -C "vmware@tkg" -f ~/tkg
```
Both public and private keys are created on the home folder under the name tkg and tkg.pub

### Deploy the management cluster
1. On the bootstrap VM, execute the following command :
  ```bash
  tanzu management-cluster create --ui
  ```
2. Access the UI with a web browser http://localhost:8080/#/ui (either on your machine or on the bootstrap VM. See Section [SSH Port Forwarding](#ssh-port-forwarding))
3. Fill in the vCenter information and credentials, as well as the public key you created on section [SSH Key](#ssh-key)
   1. Select **Deploy TKG Management Cluster**
4. Click on the tile you want (either Development or Production), and select the Instance Type. Select NSX Advanced Load Balancer as the **Control Plane Endpoint Provider**
  ![](images/tkg-deploy-01.png)
5. Insert the FQDN of the AVI Controller VIP, the username and password. Add the Root CA certificate that was used to sign the AVI controller certificate in section [Certificate](#certificate)
   1. Click on Verify
   2. Select the Cloud we created, the Default SE and the Front-End Network
    ![](images/tkg-deploy-02.png)
6. Add Metadata (not mandatory)
7. Select the VM Folder where the K8s VMs will be deployed, as we as the Datastore and resource pool
   ![](images/tkg-deploy-03.png)
8. Select the Network that will be used for Kubernetes VMs. **It's our Management K8s network here, the one that requires a DHCP**
   ![](images/tkg-deploy-04.png)
9. Choose if you want to have LDAP or OIDC authentication here.
   1.  For LDAPS, find the instruction bellow :
       1.  Fill in the LDAPS Endpoint, BIND DN, BIND Password, **Username (UserPrincipalName)**, **User Attribute (UserPrincipalName)**, and RootCA. Other are optional
      ![](images/tkg-deploy-05.png)
10. Select the OS image you want to use
11. (Optional) Register with TMC
12. Click on **Review Configuration**

**For LDAPS Identity Management, It's important to specify UserPrincipalName, even if optional. If you don't do it, the deployment of Pinniped won't work**

When you clicked on **Review Configuration**, it automatically created the YAML file that contains all the information you provided via the UI.
We are going to use it to deploy the management cluster with more verbosity.
Copy the CLI command at the bottom of the page and paste it and execute it on your bootstrap VM.

Example :
```bash
tanzu management-cluster create --file /home/vmware/.config/tanzu/tkg/clusterconfigs/201puronh8.yaml -v 6
```

Once you execute this command, it's going to ask you to questions:
1. Reply N for the first one (*Do you want to configure vSphere with Tanzu?*)
2. Reply Y for the second (*Would you like to deploy a non-integrated Tanzu Kubernetes Grid management cluster on vSphere 7.0? [y/N]*)

## Tanzu Kubernetes Grid Post Deployment

After the management cluster has been deployed, you need to execute some commands to be able to connect to it. Especially when you have deployed it with OIDC or LDAPS (Pinniped)

### Authenticate as admin on the Management Cluster

1. Get the admin context 
  ```bash
  tanzu management-cluster kubeconfig get tkg-mgmt --admin
  ```
2. The command line will return the command to authenticate to the management cluster as admin (creation of a Kubernetes context)
3. Execute this command, e.g.
  ```bash
  kubectl config use-context tkg-mgmt-vsphere-20220106172239-admin@tkg-mgmt-vsphere-20220106172239
  ```

### Verify that all apps where deployed successfully 

1. After connecting to the management cluster as admin ([Connect to the Management Cluster](#authenticate-as-admin-on-the-management-cluster))
2. Verify that all apps are reconciled successfully
  ```bash
  kubectl get apps -A
  ```

### Pinniped

After deploying the management cluster, we need to create a load balancer service for Pinniped.

1. Create a file pinniped-supervisor-svc-overlay.yaml with the following content:
  ```yaml
  #@ load("@ytt:overlay", "overlay")
  #@overlay/match by=overlay.subset({"kind": "Service", "metadata": {"name": "pinniped-supervisor", "namespace": "pinniped-supervisor"}})
  ---
  #@overlay/replace
  spec:
    type: LoadBalancer
    selector:
      app: pinniped-supervisor
    ports:
      - name: https
        protocol: TCP
        port: 443
        targetPort: 8443

  #@ load("@ytt:overlay", "overlay")
  #@overlay/match by=overlay.subset({"kind": "Service", "metadata": {"name": "dexsvc", "namespace": "tanzu-system-auth"}}), missing_ok=True
  ---
  #@overlay/replace
  spec:
    type: LoadBalancer
    selector:
      app: dex
    ports:
      - name: dex
        protocol: TCP
        port: 443
        targetPort: https
  ```
2. Convert the file into a base64-encoded string:
  ```bash
  cat pinniped-supervisor-svc-overlay.yaml | base64 -w 0
  ```
3. Get the name of the pinniped-addon secret :
  ```bash
  kubectl get secrets -n tkg-system | grep pinniped-addon
  ```   
4. Patch the mgmt-pinniped-addon secret, which contains the Pinniped configuration values, with the overlay values (replace mgmt-pinniped-addon with the result of step 3 ; replace OVERLAY-BASE64 with the output of the step 2):
  ```bash
  kubectl patch secret mgmt-pinniped-addon -n tkg-system -p '{"data": {"overlays.yaml": "OVERLAY-BASE64"}}'
  ```
5. After a few seconds, list the pinniped-supervisor (and dexsvc if using LDAP) services to confirm that they now have type LoadBalancer:
  ```bash
  kubectl get services -n pinniped-supervisor
  ```
6. Delete pinniped-post-deploy-job to re-run it:
  ```bash
  kubectl delete jobs pinniped-post-deploy-job -n pinniped-supervisor
  ```
7. Wait for the Pinniped post-deploy job to re-create, run, and complete, which may take a few minutes. You can check status by kubectl get job:
  ```bash
  kubectl get job pinniped-post-deploy-job -n pinniped-supervisor
  ```

### Configure DNS Delegation (NSX ALB or External-DNS)

#### NSX ALB (Enterprise Edition)
If you use NSX ALB with Enterprise edition, you can enable the DNS service on it, and delegate part of the domain. 
The purpose is that it'll automatically create the DNS record required when using Ingress services.

Follow these steps to enable the DNS Service on AVI
1. Go to **Administration > Settings > DNS Service > Create Virtual Service**
2. Specify a name, the network where the VIP of the DNS server will reside (*mgmt-k8s-ingress*), and its IPv4 subnet. Check the "Ignore network reachability constraints for the server pool" box
  ![](images/dns-delegation-avi-01.png)
3. Click on Next on all other pages without changing the default configuration
4. Verify the VIP that was choosen for the DNS Service, **Applications > VS VIPs**
  ![](images/dns-delegation-avi-02.png)

#### External-DNS
*This section needs to be completed*

#### Windows DNS

Once the DNS server is created on AVI, or that external-dns is configured, we need to create a new zone with delegation on windows DNS :
1. Open the DNS Manager
2. Extend the Forward Look Zone where you want to create you subzone, right-click on it and select **New Delegation**
  ![](images/dns-delegation-windows-01.png)
3. Fill in the name of the subzone you want to create
  ![](images/dns-delegation-windows-02.png)
4. Add the FQDN and the IP address of the DNS server which will manage this zone (for example, the AVI VIP)

**Do not pay attention at the error message for failed validation on step 4. VIP address for DNS on AVI controller is not pingable**

### Configure NSX ALB for Workload Clusters

You can configure some parameters of NSX ALB for when it will be automatically deployed on TKG Guest clusters. 
The main scenario where you'll want to do that is when you want to use NSX ALB as the Ingress of your K8s clusters.

#### L7 Ingress controller (NSX ALB Enterprise Edition)
There are 3 ways to configure NSX ALB as the default L7 ingress for Kubernetes :
1. L7 Ingress in ClusterIP Mode
2. L7 Ingress in NodePortLocal Mode (Not yet supported by VMware)
3. L7 Ingress in NodePort Mode 

As the 2nd method using NodePortLocal mode is not supported by VMware, we won't describe it.

Methods 1 and 3 have their pros and cons. 

**L7 Ingress in ClusterIP Mode**

**Pros**
  - ClusterIP used, so no need to change parameters when deploying apps through helm charts (default mode)
  - Performance (an AVI SE for each Workload Cluster)

**Cons**
   - Each SE group can only be used by one workload cluster, so you need a dedicated AKODeploymentConfig per cluster for AKO to work in this mode.

**L7 Ingress in NodePort Mode**

**Pros**
  - An AVI SE Group can host multiple TKG Guest Clusters

**Cons**
   - The services of your workloads must be set to NodePort instead of ClusterIP even when accompanied by an ingress object. This ensures that NodePorts are created on the worker nodes and traffic can flow through the SEs to the pods via the NodePorts.

##### L7 Ingress in ClusterIP Mode

1. Make sure you use an Enterprise License on your AVI Controller
2. Create an AVI SE Group for each TKG Guest Cluster that will require AVI as the Ingress controller
   1. Go to your AVI controller, then **Infrastructure > Service Engine Group > Select Cloud > Create**
   ![](images/l7-se-group-1.png)
   ![](images/l7-se-group-1.png)
3. Set the context of kubectl to your management cluster
   ```bash
   kubectl config use-context tkg-mgmt-vsphere-20220106172239-admin@tkg-mgmt-vsphere-20220106172239
   ```
4. Create an AKODeploymentConfig YAML file for the new configuration. Set the parameters as shown in the following sample:
  ```yaml
  apiVersion: networking.tkg.tanzu.vmware.com/v1alpha1
  kind: AKODeploymentConfig
  metadata:
    name: ako-shared-svc                                          # Change name
  spec:
    adminCredentialRef:
      name: avi-controller-credentials
      namespace: tkg-system-networking
    certificateAuthorityRef:
      name: avi-controller-ca
      namespace: tkg-system-networking
    cloudName: vcf-lab-m1-vc                                      # Change Cloud
    clusterSelector:
      matchLabels:
        ako-l7-shared-svc: "true"                                   # Change LABEL
    controller: vcf-lab-lb-ctr-vip.sddc.cce.ge                    # Change IP
    dataNetwork:
      cidr: 10.30.230.64/27                                       # Use CIDR of Front-End Network
      name: mgmt-k8s-ingress                                      # Use Front-End Network
    extraConfigs:
      disableStaticRouteSync: false                               # required
      image:
        pullPolicy: IfNotPresent
        repository: projects.registry.vmware.com/tkg/ako
        version: v1.3.2_vmware.1
      ingress:
        disableIngressClass: false                                # required
        nodeNetworkList:                                          # required
          - cidrs:
              - 10.30.231.0/24                                    # Use CIDR of K8s Management Network
            networkName: demo-tkg-12                              # Use K8s Management Network
        serviceType: ClusterIP                                    # required
        shardVSSize: MEDIUM                                       # required
    serviceEngineGroup: tkg-shared-svc-se                         # Change SE Group
  ``` 
  Where LABEL and LABEL-VALUE define the label and value needed to assign this configuration to a workload cluster in a later step. For example, ako-l7-clusterip-01: "true"
5. Create the AKO Service
  ```bash
  kubectl apply -f ./ako-shared-svc.yaml
  ```
6. Label one of your workload clusters to match the selector. Do not label more than one workload cluster.
  ```bash 
  kubectl label cluster tkg-shared-svc ako-l7-shared-svc="true"
  ```
7. Set the context of kubectl to the workload cluster.
  ```bash
  kubectl config use-context tkg-shared-svc-admin@tkg-shared-svc
  ```
8. Run the following command to ensure that NodePort changed to ClusterIP. (It can take few minutes)
  ```bash
  watch "kubectl get cm avi-k8s-config -n avi-system -o=jsonpath='{.data.serviceType}'"
  ``` 
9.  Delete the AKO pod so it redeploys and reads the new configuration file.
  ```bash
  kubectl delete pod ako-0 -n avi-system
  ```
10. In the Avi Controller UI, go to **Applications > Virtual Services** to see an L7 virtual service similar to the following:

#### L4 LB Only (NSX ALB Essentials Edition)

### Configure a wildcard certificate for Ingress Controller

#### NSX-ALB as Ingress

1. Gather a wildcard certificate for the subzone you created on [Configure DNS Delegation (NSX ALB or External-DNS)](#configure-dns-delegation-nsx-alb-or-external-dns)
2. Convert the certificate and the key as base64
   1. Extracting the private key from a PFX
   ```bash
   openssl pkcs12 -in wild-tkg-mgmt.pfx -nocerts -out wild.key
   ```
   2. Decrypting the private key
   ```bash
   openssl rsa -in wild.key -out wild.key
   ```
   3. Extracting the certificate from the PFX
   ```bash
   openssl pkcs12 -in wild-tkg-mgmt.pfx -clcerts -nokeys -out wild.pem
   ```
   4. Convert to bas64 string to be used in the YAML file
   ```bash
   cat wild.pem | base64
   cat wild.key | base64
   ```
3. Create a secret with the name router-certs-default in the same namespace where the AKO pod is running (avi-system). Ensure that the secret has tls.crt and tls.key fields in the data section.
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: router-certs-default
    namespace: avi-system
  type: kubernetes.io/tls
  data:
    tls.crt: LS0tLS1CRUdJTiB...LS0=                                      
      
    tls.key: LS0tLS1CRUdJTi...tCg==
  ```
4. Add the annotation ako.vmware.com/enable-tls in the required Ingresses and set its value to true

#### Contour as ingress

## Deploying Guest Clusters

### Create KubeConfig files for Cluster Access

