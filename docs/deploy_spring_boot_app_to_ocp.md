[TOC]

# Deploy Spring Boot Application to OpenShift



## Summary

Deploy a Spring Boot Application to OpenShift to demonstrate:

- Application build he deployment process in OpenShift by OpenShift S2I. 
- Application operations in OpenShift.



DevOps process(part): code -> build -> deploy -> operate.



## Environment Setup



- Environment: 
  - OpenShift 4.6+

- Demo project: 
   - GitHub repo: https://github.com/stp-demo-minilab/spring-petclinic.git
   - Spring Boot 2.3.3.RELEASE
   - Database:
      - Local: H2 db
      - Test: MySQL
   - Java 1.8+
   - Maven 3.3+
 - Tools
    - IDE: Intellij IDEA, VS Code
    - OpenShift command line tool: `oc`



## OpenShift S2I



Advantage of OpenShift S2I:

- No Dockerfile
- No special Maven plugin to build image
- Support auto trigger build and deployment without pipeline
- Command line and API based, easy integrated with CI/CD tools



Disadvantage of OpenShift S2I:

- Can not support complex stages in build and deployment process, e.g. code quality anaylist, automated testing, etc.



## Application Development



```bash
# clone project
git clone https://github.com/stp-demo-minilab/spring-petclinic.git

# local build
mvn clean install

# run spring boot application
mvn spring-boot:run

```



Open <http://localhost:8080> in browser to verify the application.

Try to add a new owner, and then find the owner by the last name.



> Tips: You can use mirror maven repo (e.g. Aliyun) to speed up downloading maven dependencies.



## Application Build and Deployment



### Login OpenShift

Login with user and password:

```bash
oc login -u <user> -p <password> <API_URL>
```



Or login by user and token:

- Open OpenShift web console, click current username, and then choose "Copy login command".
- Or run `oc whoami -t` to get token of current session.



### Create Project

```bash
export GUID=c5dpn

# create project
oc new-project "$GUID-petclinic" --display-name "Spring PetClinic"
```



### Create MySQL database

```bash
# create a MySQL ephemeral from OpenShift template
# https://github.com/sclorg/mysql-container/blob/master/8.0/root/usr/share/container-scripts/mysql/README.md
oc get template -n openshift | grep -i mysql
oc describe template mysql-ephemeral -n openshift

# ephemeral is ONLY for test
# must use persistent in production, and plan persistent storage and data backup carefully
oc new-app mysql-ephemeral \
  --param DATABASE_SERVICE_NAME=mysql \
  --param MEMORY_LIMIT=512Mi \
  --param MYSQL_DATABASE=petclinic \
  --param MYSQL_ROOT_PASSWORD=petclinic \
  --param MYSQL_USER=petclinic \
  --param MYSQL_PASSWORD=petclinic \
  --param MYSQL_VERSION=8.0

# check dc rollout status
oc rollout status dc mysql
```



### Create Application

 

```bash

# deploy pet clinic app by s2i 
# openjdk builder image
oc get imagestream -n openshift | grep openjdk
oc get imagestreamtag -n openshift | grep openjdk

# must provide exact builder image name and tag  
# Note: The spring-petclinic demo only be verified by rhel7 image
oc new-app openjdk-11-rhel7:1.1~https://github.com/stp-demo-minilab/spring-petclinic.git \
  --context-dir=. \
  --name=spring-petclinic \
  --as-deployment-config=false
  
  
# add env vars
# TODO: use mysql secret inject deployment environments
# check profile and env var name in application*.properties
oc set env deployment spring-petclinic \
   SPRING_PROFILES_ACTIVE=mysql \
   MYSQL_URL=jdbc:mysql://mysql:3306/petclinic \
   MYSQL_USER=petclinic \
   MYSQL_PASS=petclinic
   



# Add triggers for deployment
# Deployments support triggering off of image changes and on config changes
oc set triggers deployment spring-petclinic --from-config=true
oc set triggers deployment spring-petclinic --from-image="$GUID-petclinic/spring-petclinic:latest" -c spring-petclinic


# check rollout status
oc rollout status deployment spring-petclinic


# check pod status
watch oc get pods
oc get pods -w

# check logs
oc get pods | grep petclinic
oc logs -f $(oc get pods | grep petclinic | awk '/Running/ {print $1}')


# expose to external route
oc expose svc spring-petclinic

```



### Verify Application

```bash
oc get route spring-petclinic

# Open spring petclinic in browser
# Add an owner "William Lin"
# Find owner by last name "Lin"

# login mysql by terminal
oc get pods | grep mysql
oc rsh $(oc get pods | grep mysql | awk '/Running/ {print $1}')
```



Verify new added owner in petclinic MySQL database:

```sql
-- login mysql database in mysql pod
mysql -u root -h mysql -p

petclinic

-- veirfy data

use petclinic;
show tables;

select * from owners;

-- exit
exit
exit
```



### Add Health check

Add health check with Spring Boot actuator `health` endpoint.

```bash

# https://www.openshift.com/blog/liveness-and-readiness-probes
# http://localhost:8080/actuator
# http://localhost:8080/actuator/health

# Set probes
oc set probe deployment spring-petclinic --readiness \
  --get-url=http://:8080/actuator/health \
  --period-seconds=3 \
  --initial-delay-seconds=40 \
  --success-threshold=1 \
  --failure-threshold=3 \
  --timeout-seconds=1
  
oc set probe deployment spring-petclinic --liveness \
  --get-url=http://:8080/actuator/health \
  --period-seconds=3 \
  --initial-delay-seconds=100 \
  --success-threshold=1 \
  --failure-threshold=3 \
  --timeout-seconds=1

# watch pod status
# Check RESTARTS of pods to see if probes are set properly
watch oc get pods

```



### Use mirror Maven repository (optional)

```bash
# Use Mirror Dependencies Repository to speed up downloading dependencies when build if necessary
oc set env bc spring-petclinic MAVEN_MIRROR_URL="https://maven.aliyun.com/repository/public"

# start build
oc start-build spring-petclinic

# check build logs
oc logs -f $(oc get pods | grep build | awk '/Running/ {print $1}')
```



References:

- https://www.openshift.com/blog/improving-build-time-java-builds-openshift



### Use local (cached) maven repo (optional) // TODO

Use local (cached) maven repo to speed up build process.

```bash
# TODO
```

We will demostrate this feature in OpenShift pipeline demo.



References:

- https://www.openshift.com/blog/decrease-maven-build-times-openshift-pipelines-using-persistent-volume-claim



### Configure GitHub webhook to autotrigger build and deployment

In OpenShift, open buildconfig, and find its GitHub Webhooks.

Click "Copy URL with secret".

Example:

```bash
https://<openshift-domain>:6443/apis/build.openshift.io/v1/namespaces/<namespace>/buildconfigs/spring-petclinic/webhooks/<secret>/github
```



Login GitHub.

Open the repository, and open Settings / Webhooks:

- <https://github.com/stp-demo-minilab/spring-petclinic/settings/hooks>

Click "Add webhook".

Fill in Webhook info:

- Payload URL：See above GitHub Webhook URL with secret。
- Content Type: `application/json`
- Secret：Leave it as empty.（This secret is not secret of Payload URL）
- SSL verification：Default is Enable SSL verification, check ''Disable" to skip SSL verification.
- Events：Default is “Just the push event".
- Active：Default is checked.



Need change default git branch of buildconfig as `main` to align with GitHub repository.

```yaml
source:
    type: Git
    git:
      uri: 'https://github.com/stp-demo-minilab/spring-petclinic.git'
      ref: main
```



Try to push something to `main` branch in GitHub repository, and watch if trigger build process.

```bash
watch oc get pods
```



In OpenShift, check "Triggered by" and "Git Commit" info in the new running build.



### Rollout and rollback（TODO)

Deploy a new versioned application.

Rollback to previous version when deployment failed.

In this demo, when you check in code, it will trigger auto deployment.

We will demostrate rollout and rollback operations in image deployment demo.



## Application Operations



### Scale up

Scale up `spring-petclinic` to 3 replicas.

```bash
oc scale --replicas=3 deployment/spring-petclinic

# check pods counts and status
watch oc get pods

oc get deployment/spring-petclinic
```





### Use secrets inject deploymen environments



Previous deployment use plain-text `MYSQL_USER` and `MYSQL_PASS`， you can modify them to use ref key-value in `mysql` secret.

Before:

```yaml
        env:
            - name: SPRING_PROFILES_ACTIVE
              value: mysql
            - name: MYSQL_URL
              value: 'jdbc:mysql://mysql:3306/petclinic'
            - name: MYSQL_USER
              value: petclinic
            - name: MYSQL_PASS
              value: petclinic
```

After:

```yaml
         env:
            - name: SPRING_PROFILES_ACTIVE
              value: mysql
            - name: MYSQL_URL
              value: 'jdbc:mysql://mysql:3306/petclinic'
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: database-user
            - name: MYSQL_PASS
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: database-password
```



You can modify the enviroments on OpenShift web console, or by command line.

```bash
oc edit deployment spring-petclinic
```



### Logging

In OpenShift, switch to Admin view, and open Monitoring / Logging, and then redirect to Kibana to search aggregated logs.

> TODO



OpenShift cluster need configure logging slack (EFK) firstly.

### Monitoring

In OpenShift, switch to Developer view, and open Monitoring.

> TODO



OpenShift cluster need configure monitoring slack (Prometheus + Grafana) firstly.



### Monitor JVM (Deprecated)

Use [Jolokia](https://jolokia.org/) monitor JVM in OpenShift.

> TODO



> Jolokia features are deprecated in OpenShift 4

References:

- https://developers.redhat.com/blog/2016/03/30/jolokia-jvm-monitoring-in-openshift/
- https://docs.openshift.com/online/pro/architecture/infrastructure_components/web_console.html





## Export resources as Yaml



```bash
mkdir -p openshift/yaml/mysql
mkdir -p openshift/yaml/spring-petclinic

cd openshift/yaml
```



```bash
# project specific resources
oc get all -o name \
  | grep -v pod \
  | grep -v "build.build.openshift.io" \
  | grep -v "replicaset" \
  | grep -v "replicationcontroller"


# mysql
oc get deploymentconfig.apps.openshift.io/mysql -o yaml > ./mysql/deploymentconfig-mysql.yaml
oc get service/mysql -o yaml > ./mysql/service-mysql.yaml
oc get secret mysql -o yaml > ./mysql/secret-mysql.yaml


# spring-petclinic
oc get deployment.apps/spring-petclinic -o yaml > ./spring-petclinic/deployment-spring-petclinic.yaml
oc get buildconfig.build.openshift.io/spring-petclinic -o yaml > ./spring-petclinic/buildconfig-spring-petclinic.yaml
oc get imagestream.image.openshift.io/spring-petclinic -o yaml > ./spring-petclinic/imagestream-spring-petclinic.yaml
oc get service/spring-petclinic -o yaml > ./spring-petclinic/service-spring-petclinic.yaml
oc get route.route.openshift.io/spring-petclinic -o yaml > ./spring-petclinic/route-spring-petclinic.yaml


```



Remove temp and namespace specific fields in exported yaml files.





## Clean and redeploy resources (optional)

Clean and redeploy resources by OpenShift Yaml files to show Infrasture as Code.



```bash
# delete resources under project
oc delete project "$GUID-petclinic"

# create project
oc new-project "$GUID-petclinic" --display-name "Spring PetClinic"


# deploy mysql firstly
cd openshift/yaml

oc apply -f ./mysql

# watch mysql deployment until it is ready
watch oc get pods

# and then deploy spring-petclinic
oc apply -f ./spring-petclinic

# watch spring-petclinic deployment until it is ready
watch oc get pods
```





## Tips

### OpenShift Command Completion

```bash
# oc completion --help
# Set the oc completion code for zsh[1] to autoload on startup
oc completion zsh > /path/to/_oc

# Add into ~/.zshrc
source /path/to/_oc

# Take it effective
source. ~/.zshrc
```



When type `oc` commands, you can press `tab` key to auto complete the command.



### Show namespace in command prompt

Clone `kube-ps1`:

```bash
git clone https://github.com/jonmosco/kube-ps1.git
```



For zsh:

```bash

# Add below contents intto ~/.zshrc
export KUBE_PS1_BINARY=oc
export KUBE_PS1_CONTEXT_ENABLE=false
export KUBE_PS1_SYMBOL_ENABLE=false

source /path/to/kube-ps1/kube-ps1.sh
PROMPT='$(kube_ps1)'$PROMPT

# take it effective
source ~/.zshrc
```



For bash:

```bash
# Add below contents into ~/.bashrc
export KUBE_PS1_BINARY=oc
export KUBE_PS1_CONTEXT_ENABLE=false
export KUBE_PS1_SYMBOL_ENABLE=false

source /path/to/kube-ps1/kube-ps1.sh
PS1='[\u@\h \W $(kube_ps1)]\$ '

# take it effective
source ~/.bashrc
```



### Json formating

```bash
# MacOs
# Install jq
brew install jq

# use jq format json output
curl -s http://localhost:8080/actuator/health | jq
```



### Watch

```bash
# MacOs
# Install watch
brew install watch

# use watch to monitor pod status
watch oc get pods

```



### Use Podman mange images

Podman only works in Linux now.

```bash
# Login OpenShift internal image registry
# https://console-openshift-console.apps.<openshift-domain>
oc whoami --show-console

export REGISTRY=default-route-openshift-image-registry.apps.<openshift-domain>
podman login -u $(oc whoami) -p $(oc whoami -t) ${REGISTRY}


# Manage images, align with `docker` commands
# list local images
podman images

```



### Use Skopeo inspect and copy images

```bash
skopeo inspect docker://${REGISTRY}/<namespace>/<image>
```



### Copy login command

Besides copy login command in OpenShift web console, you also can copy login command by below command:

```bash
echo "oc login --token=$(oc whoami -t) $(oc whoami --show-server)"
```



### Updating deployment

```bash
oc rollout restart deployment spring-petclinic
```



References:

- https://kubernetes.io/docs/reference/kubectl/cheatsheet/#updating-resources

## References

- [OpenShift Container Platform 4.6 Documentation](https://docs.openshift.com/container-platform/4.6/welcome/index.html)
- [Understanding OpenShift Container Platform development](https://docs.openshift.com/container-platform/4.6/architecture/understanding-development.html)
- <https://github.com/rht-labs/ubiquitous-journey>

