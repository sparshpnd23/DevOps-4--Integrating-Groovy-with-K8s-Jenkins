# DevOps-6--Integrating-Groovy-with-K8s-Jenkins

Namaste Everyone !!
In my earlier task, I had launched a server on Kubernetes using the Jenkins GUI. 

Link to that project - https://github.com/sparshpnd23/DevOps-Task-3-Integrating-Jenkins-with-Kubernetes-.git

In this project, the motive is to do the same but using the Groovy code & Kubernetes YML files. 

#What is Groovy ?

Groovy is a powerful, optionally typed and dynamic language, with static-typing and static compilation capabilities, for the Java platform aimed at improving developer productivity thanks to a concise, familiar and easy to learn syntax. It integrates smoothly with any Java program, and immediately delivers to your application powerful features, including scripting capabilities, Domain-Specific Language authoring, runtime and compile-time meta-programming and functional programming.

We know that Jenkins is a Java based application. So, here I'll integrate Groovy with jenkins to create all the Jenkins tasks through a Groovy code.



#About the Project -

The key points of the project are : 
1- Creation of a container image that has Jenkins installed using Dockerfile. 
2- On launching the image, Jenkins should auto start.
3- Creation of the following tasks in that Jenkins using Groovy code :-

Production : This task will auto download the code from github whenever any new code is pushed.

Deployment : A suitable Kubernetes pod will be launched automatically & the code will be auto deployed. Ex- If the code is in php, the program will auto detect and launch a php pod and deploy the code. The pod will be auto exposed & the data will be persistent as a dynamic PVC will be attached to ensure that no data is lost in case of container crash.

Testing : The Deployed code will be automatically tested and if the page isn't working, an email will be automatically sent to the devdeloper. The pod will be redeployed after the developer has corrected the code.

Monitoring : Since we are using a Replica Set, hence we need not worry about pod monitoring. If the pod goes down, the Replica Set will automatically relaunch it based on the labels.

**Step - 1:** I have created a container image that has Jenkins installed using Dockerfile.

The commands in Dockerfile are as follows--

I have used Centos latest version. We need to install sudo & wget because they will be furthur needed in the installation of Jenkins. Then we have installed Jenkins & the suitable jdk version. Since systemctl doesn't work in Centos, so we would require to use sudo service jenkins start command. But service command isn't avvailable in Centos latest image. So, we have installed /sbin/service We have also installed git because we'll need it later. The third last line is to give powers to jenkins to perform operations inside the container. The, we have started the Jenkins by sudo service jenkins start but we also need to start /bin/bash otherwise the container will close as soon as jenkins starts because there will be no tasks left. Hence, we also start the bash in same coomand.

Note : The bash has to be started in the same command because if there are more than one commands in the Dockerfile, then only the last command runs.

After that, we have exposed Port 8080 because Jenkins runs on port 8080.

![](/images/df.png)


Build an image from this Dockerfile using docker build -t NAME:TAG /location of Dockerfile/

Run a container from your newly built image & expose it to any available port using PAT.

    Ex- docker run -it -p 2301:8080 --name jenpro jentest:v5
    
    
**Now, before moving on to the building Jenkins tasks through Groovy code, we first need to setup some YML files for our Server deployment on Kubernetes, which will be needed later.

**Step -2:** Create a Dockerfile to launch the server. Now, when I had done this task using the Jenkins GUI earlier, I had checked the downloaded code & launched a suitable container. Here also, I'll consider 2 cases - If the downloaded code is in html, then the deployment will be created using the first image. If the code is in php, then the deployment will be created using the second image. You can also create a single image & install all the necessary programs in it, be it html, java, php or python, but here, to show that we can make a choice, I'll create two separate images for html & php.


**HTML DOCKERFILE**

    FROM centos:latest
    RUN yum install httpd -y
    COPY *.html /var/www/html/
    CMD /usr/sbin/httpd -DFOREGROUND && /dev/bash
    
**PHP DOCKERFILE**

    FROM centos:latest
    RUN yum install httpd -y
    RUN yum install php -y
    COPY *.php /var/www/html/
    CMD /usr/sbin/httpd -DFOREGROUND && /dev/bash

    
 **Step -2::** Creating a PV & PVC which will be used later by the server pod.
 
 
 **FOR HTML**
 
 Persistent Volume -
 
      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: server-pv-vol
        labels:
          type: local
      spec:
        storageClassName: manual
        capacity:
          storage: 5Gi
        accessModes:
          - ReadWriteOnce
        hostPath:
          path: "/mnt/sdr/data/website"
          
          
  Now, create a PV using the command -
       
       kubectl create -f server-pv-vol.yml
       
       
   Persistent Volume Claim -

      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: server-pv-vol-claim
      spec:
        storageClassName: manual
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi

We claim the storage from the PV by creating this PVC file using the following command -

    kubectl create -f server-pv-vol-claim.yml
    
    
    
**FOR PHP**


Persistent Volume -
 
      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: phpserver-pv-vol
        labels:
          type: local
      spec:
        storageClassName: manual
        capacity:
          storage: 5Gi
        accessModes:
          - ReadWriteOnce
        hostPath:
          path: "/mnt/sdr/data/website"
          
          
  Now, create a PV using the command -
       
       kubectl create -f phpserver-pv-vol.yml
       
       
   Persistent Volume Claim -

      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: phpserver-pv-vol-claim
      spec:
        storageClassName: manual
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi

We claim the storage from the PV by creating this PVC file using the following command -

    kubectl create -f phpserver-pv-vol-claim.yml
    
    
**Step - 3:** Creating a Deployment for the server.


**FOR HTML**

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: server-deploy
        labels:
          app: webserver
      spec:
        replicas: 5
        selector:
          matchLabels:
            app: webserver
        template:
          metadata:
            name: server-deploy
            labels:
              app: webserver
          spec:
            volumes:
              - name: server-pv-vol
                persistentVolumeClaim:
                  claimName: server-pv-vol-claim

            containers:
              - name: html-deploy
                image: server:v1
                imagePullPolicy: IfNotPresent
                volumeMounts:
                  - mountPath: "/var/log/httpd"
                    name: server-pv-vol


**FOR PHP**

      apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: phpserver-deploy
              labels:
                app: webserver
            spec:
              replicas: 5
              selector:
                matchLabels:
                  app: webserver
              template:
                metadata:
                  name: phpserver-deploy
                  labels:
                    app: webserver
                spec:
                  volumes:
                    - name: phpserver-pv-vol
                      persistentVolumeClaim:
                        claimName: phpserver-pv-vol-claim

                  containers:
                    - name: phpserver-deploy
                      image: phpserver:v1
                      imagePullPolicy: IfNotPresent 
                      volumeMounts:
                        - mountPath: "/var/log/httpd"
                          name: phpserver-pv-vol

**Step - 4:** Creating a service to expose the Pod to the  outside world using type=NodePort.

    apiVersion: v1
    kind: Service
    metadata:
      name: expose
    spec:
      type: NodePort

      selector: 
        app: webserver

      ports:
        - port: 80
          targetPort: 80
          nodePort: 2301


**Now, it's time to move on to Groovy to create our Jnekins tasks**

**Step -5:** Task -1 - PRODUCTION

This will download the code from the mentioned Github repo, check the extension to identify whether the code is in html or php. Then, it will push the suitable image to docker hub so that it can be used later.


            job("production") {
            steps {
            scm {
                  github("sparshpnd23/face-recog-by-transfer-learning", "master")
                }
            triggers {
                  scm("* * * * *")
                }
            shell("sudo cp -rvf * /sparsh")
            if(shell("ls /sparsh/ | grep html")) {
                  dockerBuilderPublisher {
                        dockerFileDirectory("/sparsh/")
                        cloud("docker")
            tagsString("server:v1")
                        pushOnSuccess(true)

                        fromRegistry {
                              url("sparshpnd23")
                              credentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                        }
                        pushCredentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                        cleanImages(false)
                        cleanupWithJenkinsJobDelete(false)
                        noCache(false)
                        pull(true)
                  }
            }
            else {
                  dockerBuilderPublisher {
                        dockerFileDirectory("/sparsh/")
                        cloud("docker")
            tagsString("phpserver:v1")
                        pushOnSuccess(true)

                        fromRegistry {
                              url("sparshpnd23")
                              credentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                        }
                        pushCredentialsId("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
                        cleanImages(false)
                        cleanupWithJenkinsJobDelete(false)
                        noCache(false)
                        pull(true)
                  }
            }
            }
            }
            
            
 TASK - 2: DEPLOYMENT
 
 This Jenkins task will deploy the code using the suitable container & the Kubernetes deployment created above.
 
         job("Deployment") {


          triggers {
            upstream {
              upstreamProjects("Production")
              threshold("SUCCESS")
            }  
          }


          steps {
            if(shell("ls /sparsh | grep html")) {


              shell("if sudo kubectl get pv server-pv-vol; then if sudo kubectl get pvc server-pv-vol-claim; then echo "volume present"; else kubectl create -f server-pv-vol-claim.yml; fi; else sudo kubectl create -f server-pv-vol.yml; sudo kubectl create -f server-pv-vol-claim.yml; fi; if sudo kubectl get deployments server-deploy; then sudo kubectl rollout restart deployment/server-deploy; sudo kubectl rollout status deployment/server-deploy; else sudo kubectl create -f web-deploy-server.yml; sudo kubectl create -f webserver_expose.yml; sudo kubectl get all; fi")       


          }


            else {


              shell("if sudo kubectl get pv phpserver-pv-vol; then if sudo kubectl get pvc phpserver-pv-vol-claim; then echo "volume present"; else kubectl create -f phpserver-pv-vol-claim.yml; fi; else sudo kubectl create -f phpserver-pv-vol.yml; sudo kubectl create -f phserverp-pv-vol-claim.yml; fi; if sudo kubectl get deployments phpserver-deploy; then sudo kubectl rollout restart deployment/phpserver-deploy; sudo kubectl rollout status deployment/phpserver-deploy; else sudo kubectl create -f web-deploy-phpserver.yml; sudo kubectl create -f webserver_expose.yml; sudo kubectl get all; fi")


            }
          }
        }

Now, you can go & check that your code will be deployed.


TASK - 3: TESTING 

This task will test the code & if the code is found corrupt, it will automatically send an email to the mentioned email of the developer.


        job("Testing") {

        steps {

            shell('export status=$(curl -siw "%{http_code}" -o /dev/null 192.168.99.102:2301); if [ $status -eq 200 ]; then exit 0; else python3 mail.py; exit 1; fi')
          }
        }
        
        
        
 **As far as the monitoring part is concerned, we need not worry as we have used Kubernetes Deploynment to launch the server. The Replica Set working behind the deployment will keep on monitoring the pods & will launch them again if they crash.**
 
 
 **Our system is ready to use**
 
 
 _Any suggestions are highly welcome._


