# Lab: Continuous Deployment by Using Red Hat OpenShift Pipelines


Create custom tasks.

Use tasks in pipelines.

Create, manage and execute pipelines.

### Outcomes
- Create Tekton tasks.
- Create Tekton pipelines.
- Use tasks in custom Tekton pipelines.
- Start and troubleshoot Tekton pipelines.


## Instructions

In this lab, you create a pipeline that builds and deploys the vertx-site application. 

The lab script generates an OpenShift secret that you use in one of the pipeline tasks.

You can test your pipeline by using the PipelineRun object in the /home/student/DO288/labs/pipelines-review/run.yaml file. 

To test the pipeline multiple times, first remove the pipeline run by using the oc delete -f run.yaml command, and then recreate the object.

### 1. Log in to Red Hat OpenShift.

1.1 Log in to OpenShift as the developer user.

```bash
[student@workstation ~]$ oc login -u developer -p developer https://api.ocp4.example.com:6443
Login successful.
...output omitted...
```

1.2 Ensure that you use the pipelines-review project.
```bash
[student@workstation ~]$ oc project pipelines-review
Already on project "pipelines-review" on server "https://api.ocp4.example.com:6443".
```

### 2. In the pipeline YAML file, define a task to clone the application. 

Use the following parameters:
- Use the included git-clone task from the openshift-pipelines project.
- Pass the Git URL property from the pipeline inputs to the task.
- Pass the Git revision property from the pipeline inputs to the task.
- Configure the task to use the shared workspace.
### 3. Define the maven-task task in the task.yaml file. Use the following parameters:
- Use the registry.ocp4.example.com:8443/ubi8/openjdk-11:1.16-3 container image.
- Define the following workspaces:
    - The source workspace contains the git repository.
    - The maven_config workspace contains the settings.xml file.
- Define the app_path parameter that configures the location of the application in the workspace.
- Define a working directory that combines the source workspace path with the app_path parameter.
- Use the mvn clean package -s /path/to/settings.xml command to build the application. 
- The pipeline run mounts the settings.xml file from a secret when you start the pipeline.

    Then, create the task in Red Hat OpenShift. Finally, add the task reference to the pipeline.yaml file.

### 4. In the pipeline, configure the oc-deploy task with the following parameters:
- Use the included openshift-client task.
- Configure the task workspace. The task requires access to the built artifacts from the previous task.
- Configure the task to execute the following commands:

```bash
cd  "$(workspaces.manifest_dir.path)/$(params.MVN_APP_PATH)" || exit 1
oc new-build --name=$(params.DEPLOY_APP_NAME) \
  -l app=$(params.DEPLOY_APP_NAME) --binary=true \
  --image-stream=openshift/java:8 || echo "BC already exists"
oc start-build $(params.DEPLOY_APP_NAME) --wait=true \
--from-file=$(params.DEPLOY_ARTIFACT_NAME)
oc new-app $(params.DEPLOY_APP_NAME):latest \
--name $(params.DEPLOY_APP_NAME) || echo "application exists"
oc expose svc $(params.DEPLOY_APP_NAME) || echo "route exists"
```

### 5. In the pipeline, configure the skopeo-copy-internal task with the following parameters:
- Use the provided skopeo-copy-internal task.
- Configure the source URL to use the docker://default-route-openshift-image-registry.apps.ocp4.example.com/pipelines-review/DEPLOY_APP_NAME format.
- Configure the destination URL to use the docker://registry.ocp4.example.com:8443/developer/DEPLOY_APP_NAME format.
- Configure the skopeo image to be registry.ocp4.example.com:8443/ubi9/skopeo:9.2.

    In both URLs, use the application name (DEPLOY_APP_NAME) pipeline parameter.

### 6. Create and execute the pipeline.
6.1 Create the pipeline.
```bash
[student@workstation pipelines-review]$ oc create -f pipeline.yaml- pipeline.tekton.dev/maven-java-pipeline created
```
6.2 Execute the pipeline.
```bash
[student@workstation pipelines-review]$ oc create -f run.yaml- pipelinerun.tekton.dev/testrun created
```

6.3 Wait until the pipeline finishes.
```bash
[student@workstation pipelines-review]$ tkn p logs -f -a maven-java-pipeline
2025/06/30 11:49:42 Entrypoint initialization

2025/06/30 11:49:43 Decoded script /tekton/scripts/script-0-...
...output omitted...
Writing manifest to image destination
Storing signatures
```
6.4 Verify that the pipeline succeeded.
```bash
[student@workstation pipelines-review]$ tkn pipeline list
NAME                  AGE          LAST RUN   STARTED        DURATION   STATUS
maven-java-pipeline   1 hour ago   testrun    1 minute ago   1m18s      Succeeded
```
In a web browser, open the vertx-site-pipelines-review.apps.ocp4.example.com URL to verify that the application responds. 

Alternatively, test the application by using a curl request.
```bash
[student@workstation pipelines-review]$ curl -s \
  vertx-site-pipelines-review.apps.ocp4.example.com; echo
  <html><body><h1>Welcome to your Vert.x v1.0 application!</h1></body></html>
```




