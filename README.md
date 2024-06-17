# Kubernetes CI CD Pipeline

>[!Note]
>### Pipeline Path
>Github -> Jenkins Master Server [Maven -> Sonarqube -> Ansible (Master)] -> Ansible Slave Server [Kubernetes and Docker]

>[!Important]
>Master Server will be a one server setup and we can create many ansible slave servers with kubernets and docker !!!

>[!TIP]
>Install JAVA and Python same version on all systems
>Setup Environment Variables on all servers
>Install Git, Zip on all servers started

## Jenkins Server - Master
1. Create a EC2 Server with all TCP allowed
2. Install Jenkins using official documents -> start the jenkins ```systemctl enable jenkins``` and ```systemctl start jenkins```
3. Setup jenkins plugins (make sure github, maven and ansible plugins installed)
4. Install maven using official document using wget maven download link<br />
  a. unzip -> ```sudo mv mavenfolder /opt/maven```<br />
  b. copy maven path (here /opt/maven)<br />
  c. copy java path from this script ```update-alternatives —config java```<br />
  d. paste this and make sure app paths set ```echo $NAME```<br />
  ```
  export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64/
  export ES_JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64/
  export PATH=$PATH:$JAVA_HOME/
  export M2_HOME=/opt/maven
  export PATH=$M2_HOME/bin:$PATH
  ````
5. Install Ansible using official documents and setup<br />
    a. /etc/ansible/ansible.cfg is the config file and /etc/ansible/hosts is the host info file<br />
    b. Open /etc/ansible/hosts -> Add -> ```[dev] Node1 ansible_host=13.126.158.235 ansible_user=ubuntu ansible_ssh_private_key_file=~./key.pem```<br />
    c. Must put Selva_Linux_April_2024.pem inside /var/lib/jenkins/*anywhere*/key.pem<br />
    d. Check ```ansible -m all ping```  -> has to be success<br />
    e. ```chown jenkins:jenkins -R /var/lib/jenkins/*anywhere*/key.pem```<br /> 
    f. ```chmod 600 /var/lib/jenkins/*anywhere*/key.pem```[```sometime chown jenkins:jenkins -R key.pem```] is needed <br /> 
    g. FYI : No need to give ansible path to jenkins it may rise some issues
    i. Open /etc/ansible/ansible.cfg and paste
     ```
     [defaults]
     host_key_checking = False
     ```
    h. save the key in jenkins folder or else jenkins will raise permission denied issue
## Sonarqube Server
1. Create Instance with min t2.medium -> Install postgres ```sudo apt install postgresql```
2. Login to postures using :  ```psql -h sonarqubedb.c4gge0fgnzfq.ap-south-1.rds.amazonaws.com -U postgres``` -> Enter password
3. Execute below commands by one by one inside postures
   ``` 
   a. CREATE USER sonar;
   b. ALTER USER sonar WITH ENCRYPTED PASSWORD ''selvamax'’;
   c. CREATE DATABASE sonarqube;
   d. ALTER DATABASE sonarqube OWNER TO sonar;
   e. GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;```
4. Install sonarqube latest LTS version. Now 9.9.  wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.1.69595.zip
5. unzip -> move to /opt/sonarqube
6. ```sudo vi /opt/sonarqube/conf/sonar.properties``` (give rds username, rds password, postgres link : jdbc:postgress://rds_enpoing:port_number/
7. In the same file, un # sonar.web.port, sonar.web.host and sonar.web.context(if u want path like www.google.com/map)
8. ```sudo chown -R ubuntu:ubuntu /opt/sonarqube/```
9. ```sudo sysctl -w vm.max_map_count=300000``` (increase vram size for sonarqube)
10. Execute  sh /opt/sonarqube/bin/linux-x86-64/sonar.sh start
11. Check Status  sh /opt/sonarqube/bin/linux-x86-64/sonar.sh status (check 2min after starting)
12. Once running ->  http://sonar_server_ip:9000 -> admin, admin
13. Account -> Security -> Create token for maven(global) -> Add to Jenkins Credentials and pass it to Sonarqube Scanner plugin

## Ansible Slave Server
1. Create a EC2 Server for Ansible Slave (we can create as many and attach to Ansible) with all TCP allowed
2. Install docker -> ```sudo apt-get docker.io -y``` -> Start the docker ```systemctl start docker``` -> ```sudo docker login```
3. Kubernetes -> Install kops -> install kubectl using official documentation (sometime kubectl has to download and install as ./kubectl)
4. Install AWSCLI -> ```aws configure``` -> paste credentials and check using ```aws sts get-caller-identity```
5. ```KOPS_STATE_STORE=s3://bucket_name NAME=selva.k8s.local```
6. ```ssh-keygen```
7. ```kops create cluster —zones=ap-south-1a $NAME```
8. Copy all suggestions -> run the recommended command -> again copy the suggestions -> check validation

>[!Note]
> # Tool configurations has been completed !!!

## Back to Jenkins Server (Master)
1. Copy the GitHub project fork/git link
2. Create a pipeline project -> git -> paste the repo -> turn on github webhook
3. Pipeline script -> everything should be inside node {} before that we create some files
4. Create a Dockerfile and save it in the project folder (dont include # comments)
   ```
   FROM tomcat:9 #install tomcat server
   COPY newapp.war /usr/local/tomcat/webapps/
   # move the maven build app to tomcat server
   ```
5. Create ansible playbook (playbook.yaml) and save it in project folder
   ```
   - hosts: dev
   become: yes
   tasks:
    - name: Copy War File
      ansible.builtin.copy:
        src: /var/lib/jenkins/workspace/Kubernetes-CICD/target/newapp.war
        dest: /home/ubuntu/selva/
        mode: '0777'

    - name: Copy Docker File
      ansible.builtin.copy:
        src: /var/lib/jenkins/workspace/Kubernetes-CICD/Dockerfile
        dest: /home/ubuntu/selva/
        mode: '0777'

    - name: Build Docker image
      command: docker build -t selvamax/kubecicd /home/ubuntu/selva/

    - name: Push Docker image
      command: docker push selvamax/kubecicd

    - name: Delete existing Kubernetes deployment if exists
      shell: |
        kubectl delete deployment tomcat-deployment --ignore-not-found=true
      environment:
        KUBECONFIG: /home/ubuntu/.kube/config
      args:
        executable: /bin/bash

    - name: Set KUBECONFIG environment variable and deploy resources
      shell: export KUBECONFIG=/home/ubuntu/.kube/config && kubectl create -f /home/ubuntu/selva/deployment.yaml
      args:
        executable: /bin/bash
      environment:
        KUBECONFIG: /home/ubuntu/.kube/config
   ```
6. Create a deployment.yaml and save it in Ansible Slave Server (/home/ubuntu/selva)
   ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: tomcat-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: tomcat
      template:
        metadata:
          labels:
            app: tomcat
        spec:
          containers:
          - name: tomcat
            image: selvamax/kubecicd
    ```
7. Create a service.yaml and save it in Ansible Slave Server (/home/ubuntu/selva) -> Run (one time only)
   ```
    apiVersion: v1
    kind: Service
    metadata:
      name: tomcat-service
    spec:
      type: NodePort
      selector:
        app: tomcat
      ports:
      - port: 8080
        targetPort: 8080
        nodePort: 30018
    ```
8. Paste this pipeline script
   ```
     node{
      stage('SCM Checkout'){
          git 'https://github.com/Selvamaz/Kubernetes-CICD.git'
      }
      stage('Compile-Package'){
         def mvnHome =  tool name: 'M2_HOME', type: 'maven'   
         sh "${mvnHome}/bin/mvn clean package"
         sh 'mv target/myweb*.war target/newapp.war'
      }
      stage('SonarQube Analysis'){
         def mvnHome =  tool name: 'M2_HOME', type: 'maven'
         withSonarQubeEnv('sonar'){
             sh "${mvnHome}/bin/mvn sonar:sonar"
         }
      }
      stage('Ansible Deployment'){
          sh 'ansible-playbook /etc/ansible/playbook.yaml -u deploy'
      }
    }
   ```
9. Run the build
10. Check slave machine for the file, deployment and pod.
11. Check for http://kube_node:nodeport/app_name

>[!Note]
>```ansible-playbook /var/lib/jenkins/workspace/ansible.yaml  -u deploy``` # manually run playbook in master server
>
>```ansible all -m ping``` #code to check node connections
