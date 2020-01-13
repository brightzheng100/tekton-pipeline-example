# Tekton Pipeline Example

This is a very simple but workable Tekton Pipeline which can achieve a typical "build-push-scan-deploy" process.

Copied from https://github.com/kabanero-io/collections/tree/master/incubator/common/pipelines/default with necessary modifications.


## Prerequisites

Assuming you already have Tekton set up.

For example, you have successfully installed Tekton core components in `tekton-pipelines` namespace:

```
$ kubectl get pod -n tekton-pipelines
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-dashboard-6cb44d69bc-trqcr              1/1     Running   0          5d
tekton-pipelines-controller-556d8f4494-x56fv   1/1     Running   0          5d
tekton-pipelines-webhook-849cff5cf-q5lnc       1/1     Running   0          5d
```

## Set Up the Pipeline

Check out this repo and install the `Pipeline` and `Task`s, together with necessary RBAC objects:

```
$ git clone https://github.com/brightzheng100/tekton-pipeline-example.git
$ cd tekton-pipeline-example

$ kubectl apply -f .
pipeline.tekton.dev/pipeline-build-push-deploy created
serviceaccount/sa-pipeline unchanged
role.rbac.authorization.k8s.io/role-pipeline unchanged
rolebinding.rbac.authorization.k8s.io/pipeline-role-binding unchanged
task.tekton.dev/task-build-push created
task.tekton.dev/task-deploy created
task.tekton.dev/task-image-scan created
```

## Run It

There are two major ways to run the pipeline:
- By creating `PipelineRun`
- By using Tekton CLI `tkn pipeline start`

Either way you have to define the `PipelineResource` required.

I already created a sample `PipelineResource`, please change it accordingly if you want.

```
$ kubectl apply -f run/pipelineresource-git-image.yaml
pipelineresource.tekton.dev/pr-git-source-frontend created
pipelineresource.tekton.dev/pr-docker-image-frontend created
pipelineresource.tekton.dev/pr-git-source-backend created
pipelineresource.tekton.dev/pr-docker-image-backend created
```

### By Creating `PipelineRun`s

I already created some sample `PipelineRun`s.

```
$ kubectl create -f run/pipelinerun-build-push-deploy-backend.yaml
$ kubectl create -f run/pipelinerun-build-push-deploy-frontend.yaml
```

You may use `kubectl port-forward` to access the Tekton UI:

```
$ kubectl port-forward service/tekton-dashboard -n tekton-pipelines 9097:9097
Forwarding from 127.0.0.1:9097 -> 9097
Forwarding from [::1]:9097 -> 9097
```

And then open your browser and access: http://localhost:9097

![Tekton UI](/screenshots/tekton-ui.png)

> Note: Please note that you can do the same for `Task`s by creating `TaskRun`s. This is a recommended way to debug the tasks before running the pipeline

### By Using Tekton CLI `tkn pipeline start`

Tekton CLI `tkn` might be another good way you prefer:

```
$ tkn pipeline start pipeline-build-push-deploy \
-s sa-pipeline \
-r git-source=pr-git-source-backend \
-r docker-image=pr-docker-image-backend
```

> Note: Please note that you can do the same for `Task`s by using `tkn task start`:
```
$ tkn task start task-deploy \
-s sa-pipeline \
-i git-source=pr-git-source-frontend \
-i docker-image=pr-docker-image-frontend
```
