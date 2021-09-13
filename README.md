# Wolt Assignment 

Devops Assignment 1.2
Question - The task is to write the Kubernetes deployment manifests to run Docker registry in Kubernetes with at least the following - deployment, service, persistent volume claim, garbage collect cron job, ingress, secret (if needed).
Make sure Docker registry uses Redis for cache and you run 2 replicas of Docker registry for redundancy.
Record any issues you encounter. You’re also welcome to list improvement ideas for the service that are outside the scope of this assignment.


Solution - 
- File purpose and properties    

config.yml -  Docker private registry configuration file 

registry-config-cm.yaml   - Config map for private registry to pass config.yml file 

registry-deploy-redis-sidecar.yaml  - Private registry deployment with 2 replicas and using persistent volume of PV volume and registry image env variables with redis cache. It consist Multicontainer pod with registrycontainer exposed at 5000 and Redis sidecar at localhost 6379

registry-pv.yaml  - Persistent Volume to be used by Private registry to store the values

registry-pv-claim.yaml    - Persistent volume claim used by Private registry container 

registry-deploy.yaml   - Private registry deployment  with 2 replicas and using persistent volume of PV volume. It connect to external redis which is statefulset in k8s

registry-service.yaml  - Service for private registry using nodeport type to expose port 5000 and to be utlized by load balacner 

registry-ingress-without-tls.yaml   -  Ingress resource using GKE class to provision load balancer and maped to registery-service at 5000 port without tls support.   It will privde a Public ip  address assgined via load balancer on port 80 
Public IP used for vaid for next 4 days - http://35.190.117.102:80 

registry-garbage-cronjob.yaml   - Cron job for garabage collection of registry image. Cleanup manifest and blobs of repositories images.

 
 Note - [  Kubernetes manifest has been tested in GKE Cluster in Google Cloud ] 


- Features of private registry 
It uses k8s secrets to fetch the htpasswd basic auth for Private registry basic authentication and push the images to privte registry, later on can be integrated LDAP and SSL termination.

Future addons - 
1. Can be used DNS domain name instead of IP and with ssl termination certificate 
2. Can use nginx ingress controller to have more advanced feature via label annotation like ingress authentication. 
3. Use lets encrypt to generate 
4. Docker configuration file does have more advanced features and integraton other storage backends and metrics integrate. Can use sahred nfs, s3 or Ceph distributed shared storage solution 
5. For now using Redis as sidecar proxy in container but it has some disadvantage  like multiple write of same data. Can use centralized redis cluster/statefulset in k8 to faciliate it wth more advanced redis configuration.
6. Health checks and montioring can be used for private registry. 
7. Monitoring and using k8s liveness porbes can used for deployment objects. 
8. Can use istio lateron if we follow sidecar approach and use web server to host our private registry behind proxy to enhance security 

Futher Activites 
1. Using Private registry with tls at ingress level and host mapping to private registry kubernetes service. 
-   registry-tls-deploy/   Folder container code for tls with enabled at ingress and create ssl certificate management in GKE. 
2. Using redis as centralized cluster mode with MAster slave concept and interact with private registry to overcome duplicate write redundancy
- registry-redis/    Folder contain manifest for Redis manifest deployment 
3. For now using basic auth for authentication, lateron can sit behind proxy and ldap user access.
- auth/ Folder contain htpaswd credentials used during secrets creation
4. Can use middleware like cloudfront in config.yml file of registry to help in pull image faster . For push not required.



Useful commands 

To create secret from file for basic auth and mount to private registry container
- kubectl create secret generic registry-auth --from-file=auth/htpasswd -o yaml 

 To login to single container in Pod 
 - kubectl exec -it wolt-registry-7dbf4d8887-fpgf8 -c registrycontainer -- sh

To generate htpasswd file using httpd image 
- docker run   --entrypoint htpasswd   httpd:2 -Bbn <user> <password> > auth/htpasswd
- docker run   --entrypoint htpasswd   httpd:2 -Bbn khajan khajanpassword > auth/htpasswd


Prequisite 
- To connect to private registry from your local machine 
If docker installed create file at location 
- vim /etc/docker/daemon.json 

Paste below content on it and restart docker service in local 
```
{ 
"insecure-registries": ["35.190.117.102:80"]
}
```

sudo docker service restart 

Login to docer using basic auth with insecure registry
- docker login <public-ip-registry>
- docker login 35.190.117.102:80
[ Username - khajan ]
[ PAssword - khajanpassword ]

- docker pull hello-world:latest 35.190.117.102:80
- docker tag hello-world:latest 35.190.117.102:80/hello:v1
- docker push 35.190.117.102:80/hello:v1

To run garbage collector in dry run mode [ currently used inside cronjob] 
- /bin/registry garbage-collect  --dry-run etc/docker/registry/config.yml  

List all remote repo 
- curl -u khajan:khajanpassword -i -H 'Accept:application/json' 35.190.117.102:80/v2/_catalog


To list all reposiotires in Private registry 
curl -i \
-H 'Accept:application/json' \
-H 'Authorization:Basic a2hhamFuOmtoYWphbnBhc3N3b3Jk' \
35.190.117.102:80/v2/_catalog


To List private repository tag
curl -i -H 'Accept:application/json' -H 'Authorization:Basic a2hhamFuOmtoYWphbnBhc3N3b3Jk' 35.190.117.102:80/v2/<repo-name>/tags/list


