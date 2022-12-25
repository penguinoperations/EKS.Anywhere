# EKS Anywhere guideline
This guideline is about creating vSphere EKS Anywhere Cluster.
Follow the next steps and use links.

Good luck, have fun!

1. Document all network accesses
    - Requirements:
        - `vCenter endpoint` (must be accessible to EKS Anywhere clusters)
        - `public.ecr.aws`
        - `anywhere-assets.eks.amazonaws.com` (to download the EKS Anywhere binaries, manifests and OVAs)
        - `distro.eks.amazonaws.com` (to download EKS Distro binaries and manifests)
        - `d2glxqk2uabbnd.cloudfront.net` (for EKS Anywhere and EKS Distro ECR container images)
        - `api.ecr.us-west-2.amazonaws.com` (for EKS Anywhere package authentication matching your region)
        - `d5l0dvt14r5h8.cloudfront.net` (for EKS Anywhere package ECR container images)
        - `api.github.com` (only if GitOps is enabled)

---

2. [Deploy DHCP server](https://anywhere.eks.amazonaws.com/docs/reference/vsphere/vsphere-dhcp/)
    1. STATIC IP!!!
    2. Install DHCP server
        ```sh  
        sudo apt-get install isc-dhcp-server
        ```
    3. Configure `/etc/dhcp/dhcpd.conf`
        - Update the ip address range, subnet, mask, etc to suite your configuration
        - <details><summary>important fields:</summary>

            - `option domain-name;`
            - `option domain-name-servers;`
            - `subnet;`
            - `range;`
            - `option subnet-mask;`
            - `option routers;`
                - `option domain-name-servers;`
            - <details><summary>example:</summary>

                ```
                default-lease-time 600;
                max-lease-time 7200;
                
                ddns-update-style none;
                
                authoritative;
                
                option domain-name "example.local";
                option domain-name-servers dc01.example.local, dc02.example.local;


                subnet 10.8.105.0 netmask 255.255.255.0 {
                range 10.8.105.9  10.8.105.41;
                option subnet-mask 255.255.255.0;
                option routers 10.8.105.1;
                option domain-name-servers dc01.example.local, dc02.example.local;;
                }
                ```

                </details>
        </details>
    4. Restart DHCP
        ```sh  
        service isc-dhcp-server restart
        ```

---

3. Deploy dummy(administrative machine) VM for EKS Anywhere cluster deployment
    1. [Install Docker](https://docs.docker.com/engine/install/ubuntu/)
    2. [Change Cgroup](https://anywhere.eks.amazonaws.com/docs/tasks/troubleshoot/troubleshooting/#cgroups-v2-is-not-supported-in-ubuntu-2110-and-2204) (Ubuntu)
        - edit `/etc/default/grub`
            ``` 
            GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"
            ```
    3. [Install EKS Anywhere CLI tools](https://anywhere.eks.amazonaws.com/docs/getting-started/install/)
        - install `eksctl`
            ```sh
            curl "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
                --silent --location \
                | tar xz -C /tmp
            sudo mv /tmp/eksctl /usr/local/bin/
            ```
        - Install the `eksctl-anywhere` plugin.
            ```sh
            export EKSA_RELEASE="0.13.0" OS="$(uname -s | tr A-Z a-z)" RELEASE_NUMBER=25
            curl "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/${RELEASE_NUMBER}/artifacts/eks-a/v${EKSA_RELEASE}/${OS}/amd64/eksctl-anywhere-v${EKSA_RELEASE}-${OS}-amd64.tar.gz" \
                --silent --location \
                | tar xz ./eksctl-anywhere
            sudo mv ./eksctl-anywhere /usr/local/bin/
            ```
        - Install the `kubectl` Kubernetes command line tool
            ``` sh
            export OS="$(uname -s | tr A-Z a-z)" ARCH=$(test "$(uname -m)" = 'x86_64' && echo 'amd64' || echo 'arm64')
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/${OS}/${ARCH}/kubectl"
            sudo mv ./kubectl /usr/local/bin
            sudo chmod +x /usr/local/bin/kubectl
            ```
        -  Verify eksctl anywhere version
            ```sh
            eksctl anywhere version
            ```
    4. configure vSphere User, Group, and Roles via [ekscli](https://anywhere.eks.amazonaws.com/docs/reference/vsphere/vsphere-preparation/#configure-via-eksa-cli) ***(optional)***
        - Create [user.yaml](user.yaml)
            - <details><summary>pathes</summary>

                - network path: `/<datacenter>/network/<folder>`
                - datastore path: `/<datacenter>/datastore/<folder>`
                - resourcePool path: `/<datacenter>/host/<resource-pool-name>/Resources`
                - folder path: `/<datacenter>/vm/<folder>`
                - template path: `/<datacenter>/vm/<folder>`

            </details>
        - export vSphere administrator credentials
            ```sh
            export EKSA_VSPHERE_USERNAME=<ADMIN_VSPHERE_USERNAME>
            export EKSA_VSPHERE_PASSWORD=<ADMIN_VSPHERE_PASSWORD>
            ```
        - setup vSphere
            - If the user does not already exist, you can create the user and all the specified group and role objects by running:
                ```sh
                eksctl anywhere exp vsphere setup user -f user.yaml --password '<NewUserPassword>'
                ```
            - If the user or any of the group or role objects already exist, use the force flag instead to overwrite Group-Role-Object mappings for the group, roles, and objects specified in the user.yaml config file:
                ```sh
                eksctl anywhere exp vsphere setup user -f user.yaml --force
                ```
        - **NOTE:** that there is one more manual step to configure global permissions !!!
            - vSphere does not currently support a public API for setting global permissions. Because of this, you will need to manually assign the Global Role you created to your user or group in the Global Permissions UI.


---

4. Configure vCenter
    1. Create folders:
        - Folder list tree:
            ```
            .
            └──EKS-Anywhere
               ├──Workload
               └──Templates
            ```
    2.  Create set of vSphere admin credentials to create users and groups via vCenter UI ***(optional)***
        1. Create user “eksa”
        2. Create group “eksa-group”
        3. [Create and define user roles](https://anywhere.eks.amazonaws.com/docs/reference/vsphere/vsphere-preparation/#create-and-define-user-roles) (access control/roles)
            1. Create a global custom role with privileges
                1. Define to group “eksa-group”
            2. Create a user custom role with privileges
                1. Define to user “eksa”
        4. Create a default Adminsitrator role for eks user
    3. Create Resource pool ***(optional)***

---

5. Install eks production cluster
    - Requirements:
        - Be run on an Admin machine that has certain [machine requirements](https://anywhere.eks.amazonaws.com/docs/getting-started/install/) (3.2 step)
        - Have certain [resources from your VMware vSphere deployment](https://anywhere.eks.amazonaws.com/docs/reference/vsphere/vsphere-prereq/) available. (4.1 step)
        - Have some [preparation](https://anywhere.eks.amazonaws.com/docs/reference/vsphere/vsphere-preparation/) done before creating an EKS Anywhere cluster. (4.2 step OR 3.3 step)
    1. [Create vSphere production cluster](https://anywhere.eks.amazonaws.com/docs/getting-started/production-environment/vsphere-getstarted/)
        1. Genereate an initial cluster config (named `example-cluster`)
            ```sh
            CLUSTER_NAME=example-cluster
            eksctl anywhere generate clusterconfig $CLUSTER_NAME \
               --provider vsphere > eksa-mgmt-cluster.yaml
            ```
        2. Modify the initial cluster config ([eksa-mgmt-cluster.yaml](eksa-mgmt-cluster.yaml))
            - Refer to [vsphere configuration](https://anywhere.eks.amazonaws.com/docs/reference/clusterspec/vsphere/) for information on configuring this cluster config for a vSphere provider.
            - Create at least two control plane nodes, three worker nodes, and three etcd nodes for a production cluster, to provide high availability and rolling upgrades.
        3. Set credential environment variables
            ```sh
            export EKSA_VSPHERE_USERNAME='billy'
            export EKSA_VSPHERE_PASSWORD='t0p$ecret'
            ```
        4. **Create cluster**
            ```sh
            eksctl anywhere create cluster -f eksa-mgmt-cluster.yaml
            ```

---

6. Connect EKS Anywhere to AWS Console
    1. [Register](https://eksctl.io/usage/eks-connector/) eks cluster via eksctl
        ```sh
        eksctl register cluster --name tmp-cluster --provider EKS_ANYWHERE
        ```
    2. Change [eks-connector-clusterrole.yaml](eks-connector-clusterrole.yaml) ***(optional)***
    3. Connect eks anywhere to aws console
        ```sh
        kubectl apply -f eks-connector.yaml,eks-connector-clusterrole.yaml,eks-connector-console-dashboard-full-access-group.yaml
        ```

---

7. Upgrade EKS Anywhere version
    - change version in [eksa-mgmt-cluster.yaml](eksa-mgmt-cluster.yaml)
    - Run command
        ```sh
        eksctl anywhere upgrade cluster -f eksa-mgmt-cluster.yaml
        ```

---

8. Autoscaling eks 
    - Add autoscaling [configuration](https://anywhere.eks.amazonaws.com/docs/reference/clusterspec/optional/autoscaling/)
        ```yaml
        apiVersion: anywhere.eks.amazonaws.com/v1alpha1
        kind: Cluster
        metadata:
        name: my-cluster-name
        spec:
        workerNodeGroupConfigurations:
            - autoscalingConfiguration:
                minCount: 1
                maxCount: 5
            machineGroupRef:
                kind: VSphereMachineConfig
                name: worker-machine-a
            name: md-0
            - count: 1
            autoscalingConfiguration:
                minCount: 1
                maxCount: 3
            machineGroupRef:
                kind: VSphereMachineConfig
                name: worker-machine-b
            name: md-1
        ```


## Authors

- [@patarakaci](https://github.com/patarakaci)
- [@sanny](https://www.linkedin.com/in/alexander-kvaichadze-a2256495/)