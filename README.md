# Tekton Pipeline Example

This is a very simple but workable Tekton Pipeline development process that showcases:

- How to set up Tekton Pipeline, Dashboard from scratch;
- How to reuse the predefined `Task`s from [https://github.com/tektoncd/catalog](https://github.com/tektoncd/catalog);
- How to run through the `TaskRun`s step by step to walk through a typical Tekton pipeline development process;
- How to consolidate the desired `Task`s to make it a Tekton `Pipeline`;
- How to enable `Trigger`s for more advanced event-driven CI/CD.

As a result, we're going to have a very typical Tekton-powered pipeline which includes below steps:

1. `Git` clone the configured Git repository, which is a very simple Spring Boot app [here](https://github.com/brightzheng100/spring-boot-docker), as the source;
2. Use `Maven` to build the app;
3. Use `Buildah` to build, by a given `Dockerfile`, and push the built image to target container registry;
4. Deploy it to Kubernetes.

![architecture](architecture/architecture.png)


## Prerequisites

Assuming you already have Tekton set up.

If not, you may simply run below commands to set it up in seconds:

```sh
# Tekton Pipelines
# For K8s
#$ kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.16.3/release.yaml
# For OCP, ref: https://github.com/tektoncd/pipeline/issues/3452
$ kubectl apply -f https://raw.githubusercontent.com/openshift/tektoncd-pipeline/release-v0.16.3/openshift/release/tektoncd-pipeline-v0.16.3.yaml

# Tekton Dashboard
$ kubectl apply -f https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml

$ kubectl get pod -n tekton-pipelines
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-dashboard-5675959458-xpj2q              1/1     Running   0          39s
tekton-pipelines-controller-58f8bb78fc-96zpz   1/1     Running   0          17m
tekton-pipelines-webhook-65bc86d45f-28wqt      1/1     Running   0          17m

$ tkn version
Client version: 0.15.0
Pipeline version: v0.16.3
Dashboard version: v0.14.0
```

> Note: the Tekton Pipelines version is aligned with OpenShift Pipelines in OCP v4.5.x.


## Create a Project for this demo

```sh
kubectl create namespace tekton-demo

# Or if on OpenShift
oc new-project tekton-demo
```

## Create Dependent Resources

Let's create some dependant resources:

```sh
# A PVC to host shared data
kubectl apply -f Resources/pvc.yaml -n tekton-demo

# A ConfigMap to hold the maven-settings.xml which is required by `maven` Task
kubectl create cm maven-settings --from-file=settings.xml=Resources/maven-settings.xml -n tekton-demo

# A dedicated sa for the pipeline
kubectl create sa tekton-pipeline -n tekton-demo

# Run this if it's on OpenShift
oc adm policy add-role-to-user edit -z tekton-pipeline -n tekton-demo
oc adm policy add-scc-to-user privileged -z tekton-pipeline -n tekton-demo
```

## Tasks

The cool thing about Tekton is that there are a lot of reuseable `Task`s.
You may take at look at this GitHub repo for what we can use now: https://github.com/tektoncd/catalog

I reuse some of them to build the sample pipelines, which include:

| Task  | Reference |
| ------------- | ------------- |
| git-clone  | [https://github.com/tektoncd/catalog/blob/master/task/git-clone/0.1](https://github.com/tektoncd/catalog/blob/master/task/git-clone/0.1)  |
| maven  | [https://github.com/tektoncd/catalog/tree/master/task/maven/0.1](https://github.com/tektoncd/catalog/tree/master/task/maven/0.1)  |
| buildah  | [https://github.com/tektoncd/catalog/tree/master/task/buildah/0.1](https://github.com/tektoncd/catalog/tree/master/task/buildah/0.1)  |
| kubernetes-actions  | [https://github.com/tektoncd/catalog/tree/master/task/kubernetes-actions/0.1](https://github.com/tektoncd/catalog/tree/master/task/kubernetes-actions/0.1)  |

I've already made a local copy of abovementioned `Task`s.
There are some other handmade simple `Task`s too under `/Tasks` folder.

Let's install them in one shot:

```sh
$ kubectl apply -f Tasks/ -n tekton-demo
```


## Walk it through by `TaskRun`s, step by step

TaskRuns are the way to **run** and **test** Tasks.

### 1. Git clone the repository

```sh
kubectl create -f TaskRuns/taskrun-git-clone.yaml -n tekton-demo
tkn tr logs -f -a $(tkn tr ls | awk 'NR==2{print $1}')
```

> Tip: a trick to always getting the last TR logs: `tkn tr logs -f -a $(tkn tr ls | awk 'NR==2{print $1}')`

Or

```sh
tkn task start git-clone \
    --param=url=https://github.com/brightzheng100/spring-boot-docker \
    --param=deleteExisting="true" \
    --workspace=name=output,claimName=shared-workspace \
    --showlog
```

### 2. List the directory of the cloned repository

```sh
kubectl create -f TaskRuns/taskrun-list-dir.yaml -n tekton-demo
tkn tr logs -f -a $(tkn tr ls | awk 'NR==2{print $1}')
```

Or

```sh
tkn task start list-directory \
    --workspace=name=directory,claimName=shared-workspace \
    --showlog
```

> OUTPT:
```
TaskRun started: list-directory-run-kvv4g
Waiting for logs to be available...
[list-directory] total 56
[list-directory] -rwxr-xr-x    1 10006500 99            5006 Feb 17 07:16 mvnw.cmd
[list-directory] -rwxr-xr-x    1 10006500 99            7058 Feb 17 07:16 mvnw
[list-directory] drwxr-xr-x    2 10006500 99            4096 Feb 17 07:16 k8s
[list-directory] -rwxr-xr-x    1 10006500 99            2260 Feb 17 07:16 gradlew.bat
[list-directory] -rwxr-xr-x    1 10006500 99            5299 Feb 17 07:16 gradlew
[list-directory] drwxr-xr-x    3 10006500 99            4096 Feb 17 07:16 gradle
[list-directory] -rwxr-xr-x    1 10006500 99            1298 Feb 17 07:16 build.gradle
[list-directory] -rw-r--r--    1 10006500 99             290 Feb 17 07:16 README.md
[list-directory] -rwxr-xr-x    1 10006500 99             256 Feb 17 07:16 Dockerfile
[list-directory] drwxr-xr-x    4 10006500 99            4096 Feb 17 07:16 src
[list-directory] -rwxr-xr-x    1 10006500 99            3012 Feb 17 07:16 pom.xml
```

### 3. Build the source code by `maven`

```sh
kubectl create -f TaskRuns/taskrun-maven.yaml -n tekton-demo
tkn tr logs -f -a $(tkn tr ls | awk 'NR==2{print $1}')
```

Or

```sh
tkn task start maven \
    --param=GOALS="-B,-DskipTests,clean,package" \
    --workspace=name=source,claimName=shared-workspace \
    --workspace=name=maven-settings,config=maven-settings \
    --showlog
```

### 4. List the directory of the target directory

Here we can reuse the previous `TaskRun` in step 2 with a sub folder specified:

```sh
tkn task start list-directory \
    --workspace=name=directory,claimName=shared-workspace \
    --param=sub-dirs=target \
    --showlog
```

> OUTPT:
```
TaskRun started: list-directory-run-kvkt9
Waiting for logs to be available...
[list-directory] total 15952
[list-directory] drwxr-xr-x    3 10006500 99            4096 Feb 17 07:51 maven-status
[list-directory] drwxr-xr-x    3 10006500 99            4096 Feb 17 07:51 generated-sources
[list-directory] drwxr-xr-x    3 10006500 99            4096 Feb 17 07:51 test-classes
[list-directory] drwxr-xr-x    3 10006500 99            4096 Feb 17 07:51 generated-test-sources
[list-directory] drwxr-xr-x    3 10006500 99            4096 Feb 17 07:51 classes
[list-directory] -rw-r--r--    1 10006500 99            3271 Feb 17 07:51 spring-boot-docker-0.1.0.jar.original
[list-directory] drwxr-xr-x    2 10006500 99            4096 Feb 17 07:51 maven-archiver
[list-directory] -rw-r--r--    1 10006500 99        16224714 Feb 17 07:52 spring-boot-docker-0.1.0.jar
[list-directory] drwxr-xr-x    5 10006500 99            4096 Feb 17 07:52 dependency
[list-directory] drwxr-xr-x    2 10006500 99            4096 Feb 17 07:52 dependency-maven-plugin-markers
```

### 5. Build the Docker image by `buildah`

```sh
kubectl create -f TaskRuns/taskrun-buildah.yaml -n tekton-demo
tkn tr logs -f -a $(tkn tr ls | awk 'NR==2{print $1}')
```

Or

```sh
tkn task start buildah \
    --param=IMAGE="image-registry.openshift-image-registry.svc:5000/tekton-demo/spring-boot-docker:v1.1.0" \
    --param=TLSVERIFY="false" \
    --workspace=name=source,claimName=shared-workspace \
    --serviceaccount=tekton-pipeline \
    --showlog
```

> OUTPUT:

```
TaskRun started: buildah-run-sklp6
Waiting for logs to be available...
[build] + buildah --storage-driver=overlay bud --format=oci --tls-verify=false --no-cache -f ./Dockerfile -t image-registry.openshift-image-registry.svc:5000/tekton-demo/spring-boot-docker:v1.1.0 .
[build] STEP 1: FROM openjdk:8-jdk-alpine
[build] Getting image source signatures
[build] Copying blob sha256:e7c96db7181be991f19a9fb6975cdbbd73c65f4a2681348e63a141a2192a5f10
[build] Copying blob sha256:c2274a1a0e2786ee9101b08f76111f9ab8019e368dce1e325d3c284a0ca33397
[build] Copying blob sha256:f910a506b6cb1dbec766725d70356f695ae2bf2bea6224dbe8c7c6ad4f3664a2
[build] Copying config sha256:a3562aa0b991a80cfe8172847c8be6dbf6e46340b759c2b782f8b8be45342717
[build] Writing manifest to image destination
[build] Storing signatures
[build] STEP 2: VOLUME /tmp
[build] STEP 3: ARG DEPENDENCY=target/dependency
[build] STEP 4: COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib
[build] STEP 5: COPY ${DEPENDENCY}/META-INF /app/META-INF
[build] STEP 6: COPY ${DEPENDENCY}/BOOT-INF/classes /app
[build] STEP 7: ENTRYPOINT ["java","-cp","app:app/lib/*","hello.Application"]
[build] STEP 8: COMMIT image-registry.openshift-image-registry.svc:5000/tekton-demo/spring-boot-docker:v1.1.0
[build] --> 6f68457201c
[build] 6f68457201cffe0126741877218c94d6f6ed9ca1e10e171d9d53a4a8f3a47d95

[push] + buildah --storage-driver=overlay push --tls-verify=false --digestfile /workspace/source/image-digest image-registry.openshift-image-registry.svc:5000/tekton-demo/spring-boot-docker:v1.1.0 docker://image-registry.openshift-image-registry.svc:5000/tekton-demo/spring-boot-docker:v1.1.0
[push] Getting image source signatures
[push] Copying blob sha256:13fe6c35bd383a61e330e5813e3102ea5d63aca7116b0c508e50d4ef3ad2a63c
[push] Copying blob sha256:9b9b7f3d56a01e3d9076874990c62e7a516cc4032f784f421574d06b18ef9aa4
[push] Copying blob sha256:f1b5933fe4b5f49bbe8258745cf396afe07e625bdab3168e364daf7c956b6b81
[push] Copying blob sha256:ceaf9e1ebef5f9eaa707a838848a3c13800fcf32d7757be10d4b08fb85f1bc8a
[push] Copying config sha256:6f68457201cffe0126741877218c94d6f6ed9ca1e10e171d9d53a4a8f3a47d95
[push] Writing manifest to image destination
[push] Copying config sha256:6f68457201cffe0126741877218c94d6f6ed9ca1e10e171d9d53a4a8f3a47d95
[push] Writing manifest to image destination
[push] Storing signatures

[digest-to-results] + cat /workspace/source/image-digest
[digest-to-results] + tee /tekton/results/IMAGE_DIGEST
[digest-to-results] sha256:78a0239ee2c387a223cc5c105ff41950141cb7f946a885dd8caa1c2aa333a806
```

Let's double check whether the Docker image has been present in OCP's registry:

```sh
$ curl -k -H "Authorization: Bearer $(oc whoami -t)" "https://$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/v2/_catalog" | jq .

$ curl -k -H "Authorization: Bearer $(oc whoami -t)" "https://$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/v2/tekton-demo/spring-boot-docker/tags/list" | jq .
```

We should see something like this:

```json
{
  "repositories": [
    ...
    "tekton-pipelines/spring-boot-docker"
  ]
}

{
  "name": "tekton-demo/spring-boot-docker",
  "tags": [
    "v1.1.0"
  ]
}
```

> Note: There are some commonly used commands around images

```sh
# To list all desired images
oc get images | grep spring-boot-docker

# To delete all matched images
for h in `oc get images| grep spring-boot-docker | cut -d" " -f1`;  oc delete image "$h";
```

### 6. Deploy it

```sh
kubectl create -f TaskRuns/taskrun-kubernetes-actions.yaml -n tekton-demo
tkn tr logs -f -a $(tkn tr ls | awk 'NR==2{print $1}')
```

Or

```sh
tkn task start kubernetes-actions \
    --param=script="kubectl apply -f https://raw.githubusercontent.com/brightzheng100/tekton-pipeline-example/master/manifests/deployment.yaml; kubectl get deployment;" \
    --workspace=name=kubeconfig-dir,emptyDir=  \
    --workspace=name=manifest-dir,emptyDir= \
    --serviceaccount=tekton-pipeline \
    --showlog
```

## Set Up the Pipeline

Now that all the `Task`s and `TaskRun`s have been tested successfully, building a `Pipeline` is like playing with the Lego blocks.

So it's time to wrap everything into a Pipeline.

Please check the compiled pipeline out, from here [Pipelines/pipeline-git-clone-build-push-deploy.yaml](Pipelines/pipeline-git-clone-build-push-deploy.yaml).

And then install the `Pipeline`:

```sh
kubectl apply -f Pipelines/pipeline-git-clone-build-push-deploy.yaml -n tekton-demo
```


## Run the Pipeline

There are two major ways to run the pipeline manually, other than another popular automatic way by `Trigger`s:
- By creating `PipelineRun`
- By using Tekton CLI `tkn pipeline start`

So

```sh
kubectl create -f PipelineRuns/pipelinerun-git-clone-build-push-deploy.yaml -n tekton-demo
tkn pr logs -f -a $(tkn pr ls | awk 'NR==2{print $1}')
```

Or

```
tkn pipeline start pipeline-git-clone-build-push-deploy \
    -s tekton-pipeline \
    --param=repo-url=https://github.com/brightzheng100/spring-boot-docker \
    --param=tag-name=master \
    --param=image-full-path-with-tag=image-registry.openshift-image-registry.svc:5000/tekton-pipelines/
    --param=deployment-manifest=https://raw.githubusercontent.com/brightzheng100/tekton-pipeline-example/master/manifests/deployment.yaml \
    --workspace=name=workspace,claimName=shared-workspace \
    --workspace=name=maven-settings,config=maven-settings \
    --showlog
```

## Tekton Dashboard

You may use `kubectl port-forward` to access the Tekton UI:

```
$ kubectl port-forward service/tekton-dashboard -n tekton-pipelines 9097:9097
Forwarding from 127.0.0.1:9097 -> 9097
Forwarding from [::1]:9097 -> 9097
```

And then open your browser and access: http://localhost:9097

![Tekton UI](/screenshots/tekton-ui.png)


## Triggers

Until now we have seen how to create `Task`s and `Pipeline`s and run them manually.

It's time to explore how to trigger it automatically using `Trigger`s, based on desired events.

### Set Up Triggers in `tekton-pipelines` namespace

```sh
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.11.2/release.yaml

# In OCP, we need to grant permissions for service accounts
oc adm policy add-scc-to-user anyuid -z tekton-triggers-controller -n tekton-pipelines
oc adm policy add-scc-to-user anyuid -z tekton-triggers-webhook -n tekton-pipelines
oc adm policy add-scc-to-user anyuid -z tekton-triggers-core-interceptors -n tekton-pipelines
```

### Typical Steps in `tekton-demo` working namespace

```sh
# RBAC
kubectl apply -f Triggers/rbac.yaml -n tekton-demo

# Create the trigger template
kubectl apply -f Triggers/trigger-template.yaml -n tekton-demo

# Create the trigger binding
kubectl apply -f Triggers/trigger-binding.yaml -n tekton-demo

# Create the event listener
kubectl apply -f Triggers/event-listener.yaml -n tekton-demo

# Create Ing; or in OCP, expose the event listener svc as route
oc expose svc/el-tekton-event-listener -n tekton-demo
```

After this, we can get back the exposed endpoint and create a Webhook in GitHub Repo:
1. Within the repo, click **Settings** -> **Webhooks**;
2. Click **Add webhook** button at the right;
3. Fill up the form:
   - Payload URL: <The endpoint exposed for el-tekton-event-listener, e.g. route in OCP or ingress in K8s>
   - Content type: `application/json`
   - Secret: keep it blank;
   - Which events would you like to trigger this webhook? Keep the default `Just the push event`
   - Active: keep it checked
4. Click **Add webhook** to save it.

Now, once there is a push to the repo, it will trigger the `PipelineRun` to run through the `Pipeline`, which is quite cool.


## Clean Up

To clean up `tekton-demo` namespace:

```sh
# Instances
kubectl delete tr,pr,task,pipeline,pvc,route,svc,deployment --all -n tekton-demo
kubectl delete tt,tb,el --all -n tekton-demo

kubectl delete ns tekton-demo
# or in OCP
oc delete project tekton-demo
```

To clean up `tekton-pipelines` namespace:

```sh
# Tekton Triggers
kubectl delete -f https://storage.googleapis.com/tekton-releases/triggers/previous/v0.11.2/release.yaml

# Tekton Dashboard
kubectl delete -f https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml

# Tekton Pipeline
kubectl delete -f https://raw.githubusercontent.com/openshift/tektoncd-pipeline/release-v0.16.3/openshift/release/tektoncd-pipeline-v0.16.3.yaml
```


## Further References

1. Custom image builds with Buildah:
- https://docs.openshift.com/container-platform/4.5/builds/custom-builds-buildah.html
