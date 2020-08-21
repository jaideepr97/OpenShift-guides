# OpenShift-guides
## Performing a buildpacks-v3 build on an Openshift cluster and pushing the image to an external registry 

Steps to follow:
- Install build-api operator on your openshift cluster from the Operator Hub
- ```kubectl apply  -f https://raw.githubusercontent.com/redhat-developer/build/master/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml```
  a) (NOTE:  Follow this step if you want to push your image to an external registry) Create this Secret, naming it regcred:
```shell
oc create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```
- `<your-registry-server>` is your Private Docker Registry FQDN. (https://index.docker.io/v1/ for DockerHub)
- `<your-name>` is your Docker username.
- `<your-pword>` is your Docker password.
- `<your-email>` is your Docker email.


  b) (NOTE: Follow this step if you want to push your image to an internal registry through a pod on a cluster using a service account (for example)) : Try to acquire the login credentials for your desired service account (for e.g. Builder) through your cluster's console. From the list of available secrets in your namespace, pick a builder-dockercfg secret, and expose the base64 credentials (The reveal values button on OCP console). Locate the URL for your target image registry, and copy the "auth" token. Use it to prepare a new config.json file that would look something like this:

```
{"auths": {
          <image-registry-url>: {
            "auth": <auth-token>         
           }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/19.03.8 (darwin)"
        },
        "experimental": "disabled"
}
```
Once your config.json file is ready, create this secret, naming it regcred:
```shell
oc create secret generic regcred --from-file=.dockerconfigjson=<path/to/config.json>  --type=kubernetes.io/dockerconfigjson
```
- ```kubectl apply -f build.yaml ```
- ```kubectl apply -f buildRun.yaml```

**build.yaml**:
```
apiVersion: build.dev/v1alpha1
kind: Build
metadata:
  name: buildpack-nodejs-build
spec:
  source:
    url: https://github.com/sclorg/nodejs-ex
  strategy:
    name: buildpacks-v3
    kind: ClusterBuildStrategy
  output:
    image: docker.io/jaideepr97/buildpacks-test:latest
    credentials: 
      name: regcred 

``` 

**buildRun.yaml**:
```
apiVersion: build.dev/v1alpha1
kind: BuildRun
metadata:
  name: buildpack-nodejs-buildrun
spec:
  buildRef:
    name: buildpack-nodejs-build
```
