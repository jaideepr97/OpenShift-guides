# OpenShift-guides
## Performing a kaniko build on an Openshift cluster and pushing the image to a registry (WIP)

Pre-requisites:
1. Access to an active OpenShift cluster
2. Access to a source code repository (either local or hosted somewhere fot e.g GitHub)
3. A valid Dockerfile for your target source directory (can exist anywhere, as long as a fully qualified URL is available)

Steps followed:
1. Login to your OCP cluster with ``` oc login --token=<token> --server=<server_url>``` 
and create your own project with ```oc new-project xyz```

2. (NOTE: This step is only required if you intend to push to an external image hosting registry like DockerHub, and may be skipped if you intend to push to your OCP cluster's internal image hosting registry) Create this Secret, naming it regcred:
```shell
oc create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```
- `<your-registry-server>` is your Private Docker Registry FQDN. (https://index.docker.io/v1/ for DockerHub)
- `<your-name>` is your Docker username.
- `<your-pword>` is your Docker password.
- `<your-email>` is your Docker email.

3. In your local file system, git clone your source code repository (for e.g git://github.com/openshift/golang-ex.git ) in an empty directory. Download your corresponding dockerfile from its location and place it at the root of this direcotry (if it didn't already exist there). If a specific sub-directory within your cloned repo is meant to host the code that you are trying to build an image out of (as opposed to the entire cloned directory), then place the dockerfile at the root of that sub-directory. The directory containing your source code and Dockerfile now represents your build context.

(SIDE NOTE: If using the dockerfile present in the above referenced repository, be sure to comment out the line that says "USER nobody" to avoid permission issues)

4. Once your build context is ready, compress it into a tar.gz file like so:
```shell
$ tar -czvf kaniko-build-context.tar.gz <path/to/folder>
```
5. Create an openshift-pod.yaml file that specifies 2 containers :
```
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  volumes:
  - name: build-context
    emptyDir: {}
  - name: kaniko-secret
    secret:
      secretName: regcred
      items:
        - key: .dockerconfigjson
          path: config.json
  securityContext:
    runAsUser: 0 
  serviceAccount: builder
  initContainers:
    - name: kaniko-init
      image: ubuntu
      args:
        - "sh"
        - "-c"
        - "while true; do sleep 1; if [ -f /tmp/complete ]; then break; fi done"
      volumeMounts:
        - name: build-context
          mountPath: /kaniko/build-context
  
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--dockerfile=/kaniko/build-context/golang-ex/Dockerfile"
        - "--context=dir:///kaniko/build-context/golang-ex"
        - "--destination=<image-registry>/<image-repository>:tag"
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
        - name: AWS_ACCESS_KEY_ID
          value: NOT_SET
        - name: AWS_SECRET_KEY
          value: NOT_SET
   
      volumeMounts:
        - name: build-context
          mountPath: /kaniko/build-context
        - name: kaniko-secret
          mountPath: /kaniko/.docker
```

6. Apply the pod to your OCP cluster
```shell
oc apply -f <path/to/openshift-pod.yaml> 
```
It should say "pod/kaniko created"

7. To check its status, run:
```shell
oc get pods
```
It Should say:
```shell
NAME     READY   STATUS     RESTARTS   AGE
kaniko   0/1     Init:0/1   0          50s
```

7. Copy the tar.gz created earlier from your local file system into the kaniko-init container currently running in your pod
```shell
oc cp <path/to/kaniko-build-context.tar.gz> kaniko:/tmp/kaniko-build-context.tar.gz -c kaniko-init 
```

8. From inside your kaniko-init container, extract the copied tar.gz file into the mounted path pointing to the shared volume inside your kaniko pod (so it can be accessed by other containers with access to this shared volume)
```shell
oc exec kaniko -c kaniko-init -- tar -zxf /tmp/kaniko-build-context.tar.gz -C /kaniko/build-context
```
9. SIDE NOTE: If at any point you feel like taking a closer look at what's going on inside your container (which I found to be quite useful while trying to debug the process), you can start a bash session inside your kaniko-init container and take a look:
```shell
oc exec kaniko -c kaniko-init -it /bin/bash
```

10. Once the extraction process is complete, the init container can be shut down and the kaniko container will take over beyond this point. This can be done by creating a file that serves as a trigger for this action:
```shell
oc exec kaniko -c kaniko-init -- touch /tmp/complete
```

11. At this point, if everything goes well, when you run ```oc get pods``` again, you should see:
```shell
NAME     READY   STATUS    RESTARTS   AGE
kaniko   1/1     Running   0          6m57s
```
You can also get a more detailed look at what's going on inisde the kaniko container by running:
```shell
oc describe pod kaniko
```
12. After a while, you should be able to see your pushed image reflecting in your external/internal registry
13. The pod can be deleted if needed using, ```oc delete pod kaniko```
