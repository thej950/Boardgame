# END-TO-END DevOps Workflow
-----------------------------
	1. Planing
	2. Development 
	3. CI - Continuos Integration 
	4. CD - Continuos Deploy
	5. CD - Continuos Delivery
	6. Monitoring 
  
# Scenario - Example 
	
 1. The client requested a change in the application’s background color He requeste was documentaed in JIRA ticket with detailed information 
 
 2. The Ticket was assigned to a Developer, The Develeloper completed required changes into the application code, he perform some local testing everythin fine he pushed code to GITHUB 

  -  As a DevOps i will create pipeline to automate the process 
  - Download code 
  - compile the source code for syntax error 
  - i will perform some unit testing 
  - For Code Quality check and code smells, to find bugs use SonarQube to check code vulnerability 
  - Using trivy for to find vulnerability in source code and find sensitive data and scan dependency required for application check vulnerability and outdated or any issues it shows 
  - Now build artifact with build tool 
  - publish the artifact into nexus repo so we can do proper release management version of the artifact 
  - build the dockerfile with artifact
  - scan docker image with trivy to find vulnerability
  - push docker image into dockerhub 
  - Now deploy dockerimage into k8s with manifest files 

- Monitor Application, Servers, pods 


# Jenkins Server Setup
1. Launch Server 2cpu 4gb ram 20gb storage with Installed softwares
	- openjdk-17-jdk
	- jenkins
	- maven
	- git
	- docker
	- trivy
	- kubectl 

# SonarQube Server
 - SonarQube is a very important tool for maintaining and improving code quality, by automating static code analysis, it will help developers to catch issues early ensure code is secure and efficient 
 
1. Launch Server 2cpu and 4gb ram 20gb storage with Docker (ports open 9000,22)
 
 - Run SonarQube Server with Docker container 
	
 ```bash
 docker run --name sonar-server -d -p 9000:9000 sonarqube:lts-community  
 ```
  `Access http://<public_ip>:9000
	usrname: admin
	pswd: admin`

2. Generate Token in Sonarqube 
	Administration->security->create 
	  name: sr-token
		-> copy sr-token -> add in the jenkins credentials with name "sonar-token"    

![alt text](.images/image-4.png)

![alt text](.images/image-5.png)
 
 3. Generate Webhooks in SonarQube server 
	Administration->configuration->webhooks->create->
		name: jenkins
		URL: http://<jenkins_publi_ip>:8080/sonarqube-webhook/
		
![alt text](.images/image-6.png)

# Nexus Repository Server  
 - It Provides centrolized management of artifact, including binaries, docker containers and build artifacts, it will store and retrive artifacts, it works as centralized location for managing dependencies and artifacts  
 
 1. Launch Server 2cpu and 4gb ram 20gb storage with Docker 
  - Run Nexus Server with Docker container 
	```bash
	 sudo docker run -d --name nexus -p 8081:8081 sonatype/nexus3
	```
	- Access http://<public_ip>:8081
	- Get Password container 
	
	```bash	
	docker exec -it Nexus bash
	```
		
  >> cat sonartype-work/admin.password -> copy this password 
	
 - Access from brwoser 
	usrname: admin
	pswd : ************* -> paste here copied password 


![alt text](.images/image.png)

![alt text](.images/image-1.png)


 2. ordered to publish our artifact into Nexus we have to add repository URL in our POM.XML file 
 	find-> maven-releases -> copy (http://13.232.105.162:8081/repository/maven-releases/) ->copy this 
 	find-> maven-snapshots -> copy (http://13.232.105.162:8081/repository/maven-snapshots/) ->copy this 

![alt text](.images/image-2.png)


 	Open pom.xml file find "distributionManagement" add above details like below in pom.xml file 

 ```xml
 <distributionManagement>
    <repository>
        <id>maven-releases</id>
        <url>http://13.201.64.186:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>maven-snapshots</id>
        <url>http://13.201.64.186:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
 </distributionManagement>
 ```	

![alt text](.images/image-3.png)

# Configure Jenkins Server 

 ### Plugins-Download
  1. Eclipse Temurin installer
  2. config file provider 
  3. pipeline maven integration 
  4. sonarqube scanner 
  5. docker
  6. docker pipeline
  7. kuberenetes
  8. kuberenetes cli
  9. kuberenetes client API
  10. kuberenetes credentials 
  11. maven integration 

![alt text](.images/image-7.png)

![alt text](.images/image-8.png)


 ### Add Credentials 
  - Manage Jenkins->credentials->Globals
	 1. sonarqube  
		kind: secret text 
		secret:(*****) ->paste here "sr-token" received from sonarQube token
		id: sonar-token
		description: sonar-token  

![alt text](.images/image-9.png)

![alt text](.images/image-10.png)

	 2. Add dockerhub credentials 
		usrname: navathej408
		pswd: ****
		id: docker-cred
		cred: docker-cred

![alt text](.images/msedge_qluE1Z9nVi.png)

![alt text](.images/image-12.png)


 ### Configuring Tools 
  - DashBoard->Manage Jenkins->Tools->
	 1. Java 
	   name : java17
		- select install automatically (install from adopetium.net(17 version))
	 2. SonarQube Scanner Installations
	   name : sonar-scanner 
		- select install automatically (sonarQube scanner 5.0.1.3006)
	 3. maven 
	   name : maven3 
		-> Install from Apache (version 3.6.1)
	 4. Docker 
	   name : docker 
		- select insatll automatically (latest) 

![alt text](.images/image-13.png)

 ### Configure Systems 
  - Manage Jenkins-> System-> 
	1. SonarQube Servers
	 name: sonar
	 server URL : http://<public_ip>:9000
	 server authentication token: sonar-token
    
![alt text](.images/image-14.png)

# Access SonarQube server integration from Jenkins with Pipeline stage
 - Goto Pipeline Syntax  

![alt text](.images/image-17.png)


 - Below environment need to add below tools section 

```bash
environment {
        SCANNER_HOME= tool 'sonar-scanner'  
    }
```


```bash
stage('SonarQube Analsyis') {
    steps {
        withSonarQubeEnv('sonar') {
            sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                    -Dsonar.java.binaries=. '''
        }
    }
}
```
![alt text](.images/image-18.png)

```bash
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
```

![alt text](.images/image-19.png)

![alt text](.images/image-20.png)


# To Access Nexus From Jenkins pipeline with Pipeline stage  
 - In ordered to publish an artifact to Nexus Repository we have to setup a process 
 - to access Nexus server we have to download a plugin "config file provider" this plugin we already downloaded so after this in jenkins a section available in =====>  Manage Jenkins-> Managed File  

 - open that location 

 - Manage Jenkins-> Managed File -> add a new config -> type (select Global Maven settings.xml) 
	id : (global-settings)  

 - content: --> file generated below 

 - find in the below servers section 
 - These are important to access nexus repository from jenkins 
 	```xml
	<server>
	  <id>maven-releases</id>
	  <username>admin</username>
	  <password>nexus-pswd</password>
	</server>

	<server>
	  <id>maven-snapshots</id>
	  <username>admin</username> 
	  <password>nexus-pswd</password>
	</server>
	```
  - clikc on submit

![alt text](.images/image-15.png)

![alt text](.images/image-16.png)


# Nexus integration from jenkins with pipeline stage 
 - from Pipeline Syntax (withMaven:Provide Maven Environment)

```bash
stage ("Publish to Nexus") {
    steps {
        script {
            withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'java17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
            sh "mvn deploy"
            }
        }
    }
}
```
![alt text](.images/nexus.png)

# Docker Integration From jenkins 
 - added Dockerhub credentials in jenkins For to push image  
 - create image and make tag
 - scan using trivy for to find image vulnerability 
 - push docker image into dockerhub 

```bash
stage('Build & Tag Docker Image') {
	steps {
	   script {
		   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
					sh "docker build -t navathej408/boardshack:latest ."
			}
	   }
	}
}

stage('Docker Image Scan') {
	steps {
		sh "trivy image --format table -o trivy-image-report.html navathej408/boardshack:latest "
	}
}

stage('Push Docker Image') {
	steps {
	   script {
		   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
					sh "docker push navathej408/boardshack:latest"
			}
	   }
	}
}

```

![alt text](.images/image-11.png)    








# KUBEADM SETUP 
 1. Launch 3 machines with t2.medium 2cp, 4gb ram, 20gb storage 
  - install required tools on all machines  kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1
  - Initialise Master with IP address range for pods to use 
  - Deploy Calico Network and Ingress (NGINX) on master machine
 2. Generate token on Master Paste into worker machines to work as worker machines   



# TO SCAN K8S CLUSTER OPTIONAL  
```bash
# Note: After setting up k8s cluster best practice to scan k8s cluster before using tools in this link <https://github.com/Shopify/kubeaudit/releases/>
tar -xvzf kubeaudit_linux_amd64.tar.gz
sudo mv kubeaudit /usr/local/bin/
kubeaudit all 
```

# APPLICATION DEPLOYMENT INTO K8S 
1. Deploy app into k8s cluster using with RBAC - Role Based Access 
2. Role will Conatine Required Permissions - It basically used for attached to Users like (ServiceAccount) 
  - Create webapps namespace
  - create ServiceAccount with jenkins username 
  - create Role with required permissions 
  - Bind Role to jenkins user  (Now jenkins user able to deploy app into k8s cluster)   
3. Create a Token for k8s authentication from jenkins user | [ServiceAccount token](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.)

4. after creating secret yaml file | it will give token describe it take copy of it Paste into Jenkins Dashboard -> Manage Jenkins -> Credentials with name (k8s-cred) 
```bash
kubectl describe secret mysecretname -n webapps # this command will give you token deatils 
```

```bash
################################################################
# ############### ORDERED TO CREATE ROLE #######################
################################################################
# CREATE A NAMESPACE WITH (ns.yml) file  
apiVersion: v1
kind: Namespace
metadata:
  name: webapps

# CREATE SERVICE ACCOUNT WITH jenkins USERNAME (svc.yml) file
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps


# CREATE ROLE (role.yml) file 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]


# BIND ROLE TO SERVICE ACCOUNT - jenkins USER (bind.yml) file 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 

# CREATE TOKEN (sec.yml)
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  namespace: webapps
  annotations:
    kubernetes.io/service-account.name: jenkins 
```





# Complete Pipeline 

```bash

pipeline {
    agent any
    
    tools {
        jdk 'java17'
        maven 'maven3'
    }

	# Below step only add when you add stage of sonarqube 
    enviornment {
        SCANNER_HOME= tool 'sonar-scanner'  
    }

    stages {
        stage('Git Checkout') {
            steps {
               git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/jaiswaladi246/Boardgame.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analsyis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                            -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        
        stage('Build') {
            steps {
               sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
               withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t adijaiswal/boardshack:latest ."
                    }
               }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html adijaiswal/boardshack:latest "
            }
        }
        
        stage('Push Docker Image') {
            steps {
               script {
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker push adijaiswal/boardshack:latest"
                    }
               }
            }
        }
        stage('Deploy To Kubernetes') {
            steps {
               withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.146:6443') {
                        sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        stage('Verify the Deployment') {
            steps {
               withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.146:6443') {
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                }
            }
        }
        
        
    }
    post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'jaiswaladi246@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}

}
```

# Softwares to download 
```bash
#####################################################
########### On Jenkins server setup softwares ###############
############ Jenkins and java Download ####################
##########################################################
sudo apt update -y
sudo apt install openjdk-17-jdk -y
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get install jenkins -y
# trive download 
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# On sonarqube server 
curl -fsSL https://get.docker.com -o install-docker.sh
sh install-docker.sh
docker run --name sonar-server -d -p 9000:9000 sonarqube:lts-community

# On nexus server 
curl -fsSL https://get.docker.com -o install-docker.sh
sh install-docker.sh
sudo docker run -d --name nexus -p 8081:8081 sonatype/nexus3

################################################################
##################### KUBEADM SETUP COMMANDS #################### 
########################## USE Below Link to Download Latest Version of Kubeadm #########################
#### https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ ############
########################################################################
# UPDATE SYSTEM REPO
sudo apt-get update -y

# INSTALLING DOCKER ON WORKER AND MASTER MACHINES 
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock # This command Help to provide all users able to run docker commands without sudo using  

# INSTALL REQUIRED DEPENDENCIES ON MASTER AND WORKER MACHINES 
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# ADDING K8S REPO AND GPG KEY ON MASTER AND WORKER MACHINES 
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# UPDATING REPO AGAIN ON MASTER AND WORKER MACHINES 
sudo apt-get update

# INSTALLING K8S COMPONENTS ON MASTER AND WORKER NODE 
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl



# INITIALIZING K8S-MASTER MACHINE ONLY  
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 # this ip address <10.244.0.0/16> range of network ips we get and each pod will get assigned to each ip from this range only | if want we can change range also | and this command will give a command like token this will help to connect worker machines if we paste on worker machines 

# CONFIGURE K8S ON MASTER MACHINE 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# CREATING OR DEPLOYING CALICO NETWORK ON MASTER MACHINE 
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# DEPLOY OR CREATE INGRESS CONTROLLER (NGINX) ON MASTER MACHINE 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml

```



# Pipeline Code 


```bash
pipeline {
    agent any
    tools {
        jdk 'java17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME= tool 'sonar-scanner'  
    }

    stages {
        stage('checkout') {
            steps {
			cleanWs()
            git branch: 'main', url: 'https://github.com/thej950/Boardgame.git'
            }
        }
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
		stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        stage('SonarQube Analsyis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                            -Dsonar.java.binaries=. '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        stage('Build') {
            steps {
               sh "mvn package"
            }
        }
        stage ("Publish to Nexus") {
            steps {
                script {
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'java17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                    }
                }
            }
        }
        stage('Build & Tag Docker Image') {
        	steps {
        	   script {
        		   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
        					sh "docker build -t navathej408/boardshack:latest ."
        			}
        	   }
        	}
        }
        stage('Docker Image Scan') {
        	steps {
        		sh "trivy image --format table -o trivy-image-report.html navathej408/boardshack:latest "
        	}
        }
        stage('Push Docker Image') {
        	steps {
        	   script {
        		   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
        					sh "docker push navathej408/boardshack:latest"
        			}
        	   }
        	}
        }

	}
}


```

