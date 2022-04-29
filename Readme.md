Ansible script & configuration to deploy wordpress on kubernetes
Nite: This script is been made taking the fact into consideration that an ec2 instance already been deployed with 
adequate requirement & running Ubuntu OS, As all of this is been deployed taking Ubuntu into consideration.
All the packages (ansible & kubernetes ) are deployed on localhost. No another machine is taken into consideration.
Remember this is done only for assignment purpose & is not recommended in real production scenario

Steps to follow :

STEP 1 : Ansible Installation and Configuration
STEP 2 : Create Ansible Roles
STEP 3 : Write role for Kubernetes
STEP 4 : Write role for Wordpress
STEP 5 : Create Setup file
STEP 6 : RUN your Ansible Playbook

STEP 1 : Ansible Installation and Configuration

sudo apt-get update
sudo apt-get install python
sudo apt-get install ansible

Note: Python should be installed on your OS to setup Ansible. 
Now lets access the ansible configfile & make the desired changes

Command: vim /etc/ansible/ansible.cfg

Write the below commands in the configuration file for ansible. Most of the commands are already present by default in the config file just uncomment it by removing the #

[defaults]
inventory=/etc/ansible/hosts          #inventory path
host_key_checking=False
command_warnings=False
deprecation_warnings=False
ask_pass=False
roles_path=/etc/ansible/roles      #roles path
force_valid_group_names = ignore
private_key_file=~/.ssh/demoawskey.pem          #your key-pair
remote_user=ec2-user     #default ec2 username
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False



STEP 2 : Create Ansible Roles

Go inside the roles directory

Command: cd /roles

Use Below commands to create 3 different roles

Command: 
ansible-galaxy init kube_role                  #For Kubernetes 
ansible-galaxy init wordpress                 #For Wordpress



STEP 3 : Write role for Kubernetes 

Go inside the tasks folder & create a file main.yml for the cluster

Command: 

cd /roles/kube_role/tasks
vim main.yml

Add the below code in the main.yml file for cluster

 - name: Make the Swap inactive
   command: swapoff -a

 - name: Remove Swap entry from /etc/fstab.
   lineinfile:
    dest: /etc/fstab
    regexp: swap
    state: absent

 - name: Installing Prerequisites for Kubernetes
   apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - vim
      - software-properties-common
    state: present

 - name: Add Dockers official GPG key
   apt_key:
      url: https://download.docker.com/linux/ubuntu/gpg
      state: present

 - name: Add Docker Repository
   apt_repository:
      repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable
      state: present
      filename: docker
      mode: 0600

 - name: Install Docker Engine & iproute.
   apt:
      name:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        - iproute2
      state: present

 - name: Enable service docker, and enable persistently
   service:
     name: docker
     enabled: yes

 - name: Add Google official GPG key
   apt_key:
      url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
      state: present

 - name: Add Kubernetes Repository
   apt_repository:
      repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
      state: present
      filename: kubernetes
      mode: 0600

 - name: Installing Kubernetes Cluster Packages.
   apt:
      name:
        - kubeadm
        - kubectl
        - kubelet
      state: present

 - name: Enable service kubelet, and enable persistently
   service:
      name: kubelet
      enabled: yes

 - name: "Pulling the config images"
   shell: kubeadm config images pull

 - name: "Confuring the docker daemon.json file"
   copy:
    dest: /etc/docker/daemon.json
    content: |
     {
     "exec-opts": ["native.cgroupdriver=systemd"]
     }

 - name: "Restarting the docker service"
   service:
      name: docker
      state: restarted

 - name: "Configuring the Ip tables and refreshing sysctl"
   copy:
    dest: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-ip6tables = 1
      net.bridge.bridge-nf-call-iptables = 1

 - name: "systemctl"
   shell: "sysctl --system"

 - name: "Starting kubeadm service"
   shell: "kubeadm init  --ignore-preflight-errors=all"

 - name: "Creating .kube Directory"
   file:
      path: $HOME/.kube
      state: directory

 - name: "Copying file config file"
   shell: "cp -i /etc/kubernetes/admin.conf $HOME/.kube/config"
   ignore_errors: yes

 - name: "Installing Addons e.g flannel"
   shell: "kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"



STEP 4 : Write role for Wordpress

Go inside the files folder of wordpress role. We have to write entire configuration files inside this folder.

Commands: 

cd roles/wordpress/files/

vi wordpress.yml
vi pvc_wordpress.yml

Copy the below content inside wordpress.yml

apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
      nodePort: 30333
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pv-claim
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysqlsecret
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumes:
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html



Copy the below content inside pvc_wordpress.yml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
   name: wordpress-pv-claim
   labels:
        app: wordpress
        tier: frontend
spec:
   storageClassName: ""
   resources:
        requests:
             storage: 1Gi
   accessModes:
     - ReadWriteOnce
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  storageClassName: ""
  capacity:
     storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /wordpressdata



Now we need to create tasks under the wordpress role

cd roles/wordpress/tasks/
vi main.yml

---
# tasks file for wordpress
  - name: Copying Wordpress files to K8s Master Node
    copy:
        src: "{{ item }}"
        dest: /root/
    loop:
        - pvc_wordpress.yml
        - wordpress.yml


  - name: Creating directory over which WordPress container mounts the PersistentVolume at /var/www/html.
    file:
        path: /wordpressdata
        state: directory

  - name: Configuration and Setup of Wordpress
    shell: "kubectl create -f /root/{{ item }}"
    loop:
        - pvc_wordpress.yml
        - wordpress.yml




STEP 5 : Create Setup file


- hosts: localhost
  gather_facts: no
  tasks:
    - name: Running K8 Role
      include_role:
        name: kube_role
- hosts: localhost
  gather_facts: no
  tasks:
    - name: Running Wordpress Role
      include_role:
        name: wordpress



STEP 6 : RUN your Ansible Playbook

ansible-playbook setup.yml