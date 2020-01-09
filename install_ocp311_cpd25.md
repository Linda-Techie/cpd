# Note: The baseline of OCP content originally from Victor Havard. This doc is intented to be used as a cookbook and labs. 

## Table Content
  * OCP v3.11
  * Storage: NFS + Portworx
  * CPD v2.5
  * Addon: Watson 

## Document is based on and tested with following cluster spec 

For this exercise, the following nodes will be deployed (non-HA instances will only need one of each node type):

|    Node         |  Host   | Storage      | vCPU  | Memory |
|:--------------- |:-------:|:------------ |:----: |:------:|
| ansible         | ansible | 100,100      |   4   |    8   |
| lb-infra        | lb      | 100          |   4   |    8   |
| master1+infra   | master1 | 100,200      |  16   |   64   |
| master2+infra   | master2 | 100,200      |  16   |   64   |
| master3+infra   | master3 | 100,200      |  16   |   64   |
| worker1         | node1   | 100,200,200  |   8   |   32   |
| worker2         | node2   | 100,200,200  |   8   |   32   |
| worker3         | node3   | 100,200,200  |   8   |   32   |
| worker4         | node4   | 100,200,200  |   8   |   32   |
| worker5         | node5   | 100,200,200  |   8   |   32   |


## Prepare Nodes for Installation

**Note:** The following steps should be carried out on **all** nodes and be run as the `root` user.

1. Install all nodes with Red Hat Enterprise Linux version 7.4 or higher with only the Minimal installation packages. Only install the operating system on the first allocated disk (e.g. /dev/sda).  Do not put a partition on any disk other than the disk where the operating system will be installed (the boot disk).

2. No partition on the 2nd and/or 3rd disks for all masters and workers
3. NFS server disk should have partitioned and formatted with xfs
4. Configure passwordless SSH between the ansible (installer) node and all other nodes.
  ```
  sudo ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa  # That is two single quotes
  sudo cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  sudo chmod 700 ~/.ssh
  sudo chmod 0600 ~/.ssh/authorized_keys
  ```
  Accept all of the default values

5. Copy the root id_rsa.pub key to all other nodes
  ```
  ssh-copy-id -i ~/.ssh/id_rsa.pub master1.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub master2.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub master3.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub node1.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub node2.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub node3.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub node4.cp4d.csplab.local
  ssh-copy-id -i ~/.ssh/id_rsa.pub node5.cp4d.csplab.local
  ```

6. Install Red Hat Subscriptions (all cluster nodes)
  ```
  subscription-manager repos --disable="*"  # Remove existing Subscriptions
  subscription-manager register --username=<RedHat-UserID> --password=<RedHat-Password> # Register the node with Red Hat
  subscription-manager refresh
  subscription-manager attach --pool=<poolID>  # Pool with entitlement to RHOS
  ```

7. Enable the needed yum repositories (all cluster nodes)
  ```
  subscription-manager repos --enable="rhel-7-server-rpms" --enable="rhel-7-server-extras-rpms" --enable="rhel-7-server-ose-3.11-rpms"  --enable="rh-gluster-3-client-for-rhel-7-server-rpms"
  yum update -y
  ```

8. Install needed prerequisite packages (all cluster nodes)
  ```
  yum install -y wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct glusterfs-fuse ntp libsemanage-python python-setuptools python-websocket-client nfs-utils 
  
  yum install unzip podman (ansible node only)
  ```

9. Enable ntpd (all cluster nodes)
  ```
  systemctl enable ntpd
  ```

10. Once prerequisites are installed, reboot the node (all cluster nodes)
  ```
  systemctl reboot
  ```

11. Install openshift-ansible package (on ansible node only)
  ```
  yum install -y openshift-ansible
  ```

12. Install docker (all cluster nodes except Ansible and LB)
  ```
  yum install -y docker-1.13.1
  ```

13. Configure Docker Storage (all cluster nodes except Ansible and LB)
  ```
  Note: Modify the disk based on your server accordingly (use $lsblk to identify the name of the disk)
  cat > /etc/sysconfig/docker-storage-setup <<EOF
  STORAGE_DRIVER=overlay2
  DEVS=/dev/sdb
  CONTAINER_ROOT_LV_NAME=dockerlv
  CONTAINER_ROOT_LV_SIZE=100%FREE
  CONTAINER_ROOT_LV_MOUNT_PATH=/var/lib/docker
  VG=dockervg
  EOF
  ```
  This will create a new file named `docker-storage-setup` in /etc/sysconfig.  DEVS should contain the raw device which should be used for the docker partition.  The raw disk will be configured for Logical Volume Mapping (LVM) and mounted at the location specified by `CONTAINER_ROOT_LV_MOUNT_PATH`, this specified location is the default location for the docker local registry.<br><br>The value in `DEVS` should be the second raw disk in the system (e.g. /dev/sdb or /dev/vdb).  This value should be the raw disk device and should not have a partition.

14. Enable docker to be auto-started with the system
  ```
  systemctl enable docker
  ```

15. Start docker
  ```
  systemctl start docker
  ```

16. Check to make sure docker is running properly
  ```
  rpm -V docker-1.13.1
  docker version
  ```

  The results of the above command should look something like this:

  ```
  [root@ansible ~]# rpm -V docker-1.13.1
  S.5....T.  c /etc/sysconfig/docker-storage
  .M.......    /var/lib/docker

  [root@rhos-ansible ~]# docker version
  Client:
   Version:         1.13.1
   API version:     1.26
   Package version: docker-1.13.1-91.git07f3374.el7.x86_64
   Go version:      go1.10.3
   Git commit:      07f3374/1.13.1
   Built:           Fri Feb  8 20:24:43 2019
   OS/Arch:         linux/amd64

  Server:
   Version:         1.13.1
   API version:     1.26 (minimum version 1.12)
   Package version: docker-1.13.1-91.git07f3374.el7.x86_64
   Go version:      go1.10.3
   Git commit:      07f3374/1.13.1
   Built:           Fri Feb  8 20:24:43 2019
   OS/Arch:         linux/amd64
   Experimental:    false
  ```
  If you don't get similar results, make sure docker is running using the `systemctl restart docker` command.

17. Ensure the newly created docker volume is properly mounted
  ```
  [root@m1 ~]# df -Th
  Filesystem                    Type      Size  Used Avail Use% Mounted on
  devtmpfs                      devtmpfs   32G     0   32G   0% /dev
  tmpfs                         tmpfs      32G     0   32G   0% /dev/shm
  tmpfs                         tmpfs      32G  1.1M   32G   1% /run
  tmpfs                         tmpfs      32G     0   32G   0% /sys/fs/cgroup
  /dev/mapper/rhel-root         xfs        50G  3.8G   47G   8% /
  /dev/sda1                     xfs      1014M  188M  827M  19% /boot
  /dev/mapper/rhel-home         xfs       129G   33M  129G   1% /home
  tmpfs                         tmpfs     6.3G     0  6.3G   0% /run/user/0
  /dev/mapper/dockervg-dockerlv xfs       200G   33M  200G   1% /var/lib/containers/docker

  ```

  If you do not see a volume mounted to /var/lib/docker repeat the above process.

## Install Red Hat OpenShift Enterprise (OSE)
**Note:** The following should only be done on the ansible (installer) node.

18. Edit the file at /etc/ansible/hosts and add the below stanzas making updates as is needed for your implementation
  ```
  Note: 
  a) For Portworx
     Add
      * openshift_use_crio=True
      * openshift_use_crio_only=True
     
     Change [nodes] section
     Example:
     m[1:3].cp4d-5.csplab.local openshift_node_group_name="node-config-master-infra-crio"
     openshift.cp4d-5.csplab.local openshift_node_group_name="node-config-infra-crio"
     n[1:5].cp4d-5.csplab.local openshift_node_group_name="node-config-compute-crio"

     Remove or Comment out line:
     #openshift_node_groups
  b) For NFS
     Add 
     * [nfs] section 
     * openshift_hosted_registry_storage_nfs_directory=/data #Your mount drive
     * openshift_hosted_registry_storage_nfs_options='*(rw,no_root_squash,anonuid=1000,anongid=2000)' # Don't worry about the uid & gid
  
  ### OSE Inventory File
  # For more information see: https://docs.openshift.com/container-platform/3.11/install/configuring_inventory_file.html#configuring-ansible
  # This section defines the types of nodes we will deploy
# define openshift components
[OSEv3:children]
masters
nodes
nfs
etcd
lb

# define openshift variables
[OSEv3:vars]
containerized=true
openshift_deployment_type=openshift-enterprise
openshift_hosted_registry_storage_volume_size=50Gi
openshift_docker_insecure_registries="172.30.0.0/16"
openshift_disable_check=docker_image_availability
#openshift_node_groups=[ {"name": "all-in-one", "labels": ["node-role.kubernetes.io/master=true", "node-role.kubernetes.io/infra=true", "node-role.kubernetes.io/compute=true" ]}, {"name": "node-config-master", "labels": ["node-role.kubernetes.io/master=true"]}, {"name": "node-config-infra", "labels": ["node-role.kubernetes.io/infra=true"]}, {"name": "node-config-compute", "labels": ["node-role.kubernetes.io/compute=true"]}]
oreg_url=registry.access.redhat.com/openshift3/ose-${component}:${version}
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_htpasswd_users={'ocadmin': '$apr1$CdyzN7vS$wM6gchgqURLe1A7gQRbIi0'}
ansible_ssh_user=root
os_firewall_use_firewalld=True
openshift_master_api_port=443
openshift_master_console_port=443

openshift_console_install=true
openshift_console_hostname=console.apps.cp4d-5.csplab.local

openshift_master_cluster_method=native
openshift_master_cluster_hostname=openshift.cp4d-5.csplab.local
openshift_master_cluster_public_hostname=openshift.cp4d-5.csplab.local
openshift_disable_check=docker_image_availability

# CRI-O
openshift_use_crio=True
openshift_use_crio_only=True

# NFS Host Group
# An NFS volume will be created with path "nfs_directory/volume_name"
# on the host within the [nfs] host group.  For example, the volume
# path using these options would be "/exports/registry".  "exports" is
# is the name of the export served by the nfs server.  "registry" is
# the name of a directory inside of "/exports".
openshift_hosted_registry_storage_kind=nfs
openshift_hosted_registry_storage_access_modes=['ReadWriteMany']
# nfs_directory must conform to DNS-1123 subdomain must consist of lower case
# alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character
openshift_hosted_registry_storage_nfs_directory=/data
openshift_hosted_registry_storage_nfs_options='*(rw,no_root_squash,anonuid=1000,anongid=2000)'
openshift_hosted_registry_storage_volume_name=registry
openshift_hosted_registry_storage_volume_size=200Gi

# host group for masters
[masters]
m1.cp4d-5.csplab.local
m2.cp4d-5.csplab.local
m3.cp4d-5.csplab.local

# host group for etcd
[etcd]
m1.cp4d-5.csplab.local
m2.cp4d-5.csplab.local
m3.cp4d-5.csplab.local

# load balancer
[lb]
openshift.cp4d-5.csplab.local

# The value in the square brackets should indicate the raw disk to use for GlusterFS bricks.
# # These shoudld be raw disks and have no partitions defined.  If you have multiple
# #  raw disks, use a comma separated list within each square bracket.
# [glusterfs]
# storage1.cp4d.csplab.local glusterfs_devices='[ "/dev/vdc" ]'
# storage2.cp4d.csplab.local glusterfs_devices='[ "/dev/vdc" ]'
# storage3.cp4d.csplab.local glusterfs_devices='[ "/dev/vdc" ]'

# nfs server
[nfs]
n1.cp4d-5.csplab.local

# All nodes and their respective node types
[nodes]
m[1:3].cp4d-5.csplab.local openshift_node_group_name="node-config-master-infra-crio"
openshift.cp4d-5.csplab.local openshift_node_group_name="node-config-infra-crio"
n[1:5].cp4d-5.csplab.local openshift_node_group_name="node-config-compute-crio"
  ```

19. Prepare NFS server & client as needed (Optional)

For NFS Server
  * yum install -y nfs-utils nfs-utils-lib
  * systemctl enable nfs-server
  * systemctl start rpcbind
  * systemctl start nfs-server
  * firewall-cmd --permanent --zone=public --add-service=nfs
  * firewall-cmd --permanent --zone=public --add-service=rpcbind
  * firewall-cmd --reload
  
  Prepare NFS server shared disk
  
  a) lsblk
  
  b) parted /dev/sdb mklabel gpt
  
  c) parted -a opt /dev/sdb mkpart primary xfs 0% 100%
  
  d) mkfs.xfs -f -n ftype=1 -i size=512 -n size=8192 /dev/sdb1
  
  e) Create a mount drive: # mkdir /data
  
  f) Modify /etc/fstab for reboot safe
  ```
     * blkid
     * vi /etc/fstab
     
     Example:
        [root@n1 ~]# cat /etc/fstab
           /dev/mapper/rhel-root   /                       xfs     defaults        0 0
           UUID=dd72311a-64e7-45ff-9afb-a150e5dbbaf9 /boot                   xfs     defaults        0 0      
           /dev/mapper/rhel-home   /home                   xfs     defaults        0 0
           UUID=284af079-2b6c-40af-89da-229ba79f2f52  /data  xfs  defaults,noatime    1 2
  ```   
  g) mount -a
  
  h) Edit /etc/exports with all cluster nodes. Example:
     [root@n1 ~]# cat /etc/exports
     /data 172.16.56.145(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.146(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.147(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.148(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.149(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.150(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.151(rw,sync,no_root_squash,anonuid=1000,anongid=2000) 172.16.56.152(rw,sync,no_root_squash,anonuid=1000,anongid=2000)
     
  i) Modify permission
  ```
     chown 1000:2000 /data
     chmod 777 /data
  ```
  j) Restart service
  ```
     systemctl restart nfs.service
  ```
  k) Test: 
  ```
  showmount -e nfs-server.domain.com
  ```
OCP cluster nodes
  * yum install -y nfs-utils

20. Create 3 files and run with ansible-playbook
```
  # cat setupselinux.yaml
  ---
  - name: Enable SELinux
    hosts: all
    tasks:
    - name: set selinux enforcing
      selinux: state=enforcing policy=targeted

  # cat setupseboolean.yaml
  ---
  - name: SELinux Boolean
    hosts: all
    tasks:
    - name: set seboolean virt_use_nfs
      seboolean: name=virt_use_nfs state=yes persistent=yes

  #cat setvmmaxmapcount.yaml
  ---
  - name: Setup for Elasticsearch
    hosts: all
    tasks:
    - name: set vm max map count
      shell: sysctl -w vm.max_map_count=262144; echo "vm.max_map_count=262144" >> /etc/sysctl.conf

  # ansible-playbook -i hosts setupselinux.yaml
  # ansible-playbook -i hosts setupseboolean.yaml
  # ansible-playbook -i hosts setvmmaxmapcount.yaml
  
  Reboot all nodes (except ansible node)
 ```

21. Check your installation prior to install
  ```
  [root@ansible ~]# cd /usr/share/ansible/openshift-ansible

  [root@ansible openshift-ansible]# ansible-playbook playbooks/prerequisites.yml
  ```

22. Deploy the cluster
  ```
  [root@ansible openshift-ansible]# ansible-playbook -i hosts /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml -vvv | tee install.log
  
  ```

## Configure your new cluster

### Create a DNS entry for our main OpenShift UI.

In our inventory file, we created an openshift_master_cluster_public_hostname entry and set its value to openshift.mydomain.local. Before we can login to the UI using this hostname we need a alias configured in the DNS which aliases openshift.mydomain.local to your first load balancer (master-lb.mydomain.local).

In bind9, that entry would look something like this:

```
openshift	IN	CNAME	master-lb
```

### Configure new cluster
1. ssh to master1

2. login to master1
```
   ssh master1
   oc login -u system:admin
```
3. create user "ocadmin"
```
   cd /etc/origin/master;htpasswd -b htpasswd ocadmin ocadmin
```
4. Set ocadmin to cluster's admin
```
   oc adm policy add-cluster-role-to-user cluster-admin ocadmin
```

5. Verify ocadmin login
```
   oc login --username=ocadmin --password=ocadmin --insecure-skip-tls-verify
```

### Login to OCP Console
URL: https://openshift.cp4d.csplab.local:8443


### Setting up your cluster for use

  * Login for the first time

  Before you can do anything else, you must login the first time with the `oc` command and assign a user the cluster-admin role.  ssh to one of your master nodes and execute the following command:
  ```
  oc login -u system:admin
  ```

  If it worked correctly you should be able to execute commands
  ```
  [root@master1 ~]# oc get pods
  NAME                       READY     STATUS    RESTARTS   AGE
  docker-registry-1-4h574    1/1       Running   0          2h
  registry-console-1-p7q4f   1/1       Running   0          2h
  router-1-8p9xg             1/1       Running   0          2h
  router-1-fzkl4             1/1       Running   0          2h
  router-1-r8pb5             1/1       Running   0          2h

  ```

  * Create a cluster administrator user

  If you configured an LDAP identity provider in your inventory file you will already have users available to configure and those user should be able to successfully login.

  If you used the htpasswd identity provider and have not already done so, see above for instructions on creating a new htpasswd user.

  To login to the web UI, point your browser to the hostname from the value you specified in your inventory for **openshift_master_cluster_public_hostname**, in this case: https://openshift.mydomain.local.

  If you chose a port other than 443, you will have to add the port number as well (e.g. https://openshift.mydomain.com:8443).

  Login with the credentials you just created and look around.  This is the user interface for a normal user.

  When you are done, log back out of the web UI.

  * Assign a user to be the cluster-admin

  From the command line on a master node, execute the following command:
  ```
  [root@master1 ~]# oc adm policy add-cluster-role-to-user cluster-admin vhavard
  cluster role "cluster-admin" added: "vhavard"
  ```
  Where `vhavard` is the userid of a valid user from LDAP or htpasswd.

  Now login to the web UI with your cluster admin user and see the difference from a normal user.
  
  # Install Portworx
  1. Download Software 
     Part number: CC3Y1ML
     
  2. Extract software
     ```
     tar zxvf cloudpak4data-ee-v2.5.0.0.tgz
     tar zxvf cpd-portworx.tgz
     ```
  3. Download Portworx images (~
     ```
     mkdir -p /tmp/cpd-px-images
     cd cpd/cpd-portworx
     bin/px-images.sh -d /tmp/cpd-px-images download
     ```
  4. Login to OCP Cluster via command or token
  
  5. Apply the Portworx cloud Pak for data activation on OCP nodes
     ```
     bin/px-util initialize --sshKeyFile ~/.ssh/id_rsa
     ```
  6. 
  
