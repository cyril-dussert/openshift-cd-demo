# OpenShift 3 CI/CD Demo

This repository includes the infrastructure and pipeline definition for continuous delivery using Jenkins, Nexus and SonarQube on OpenShift. On every pipeline execution, the code goes through the following steps:

1. Code is cloned from Git, built, tested and analyzed for bugs and bad patterns
2. The WAR artifact is pushed to Nexus Repository manager
3. A Docker image (_tasks:latest_) is built based on the _Tasks_ application WAR artifact deployed on JBoss EAP 6
4. The _Tasks_ Docker image is deployed in a fresh new container in DEV project
5. If tests successful, the DEV image is tagged with the application version (_tasks:6.4.0_) in the STAGE project
6. The staged image is deployed in a fresh new container in the STAGE project

The following diagram shows the steps included in the deployment pipeline:

![](https://github.com/OpenShiftDemos/openshift-cd-demo/blob/openshift-3.2/images/pipeline.png)

# Setup

Create a new project for CI/CD components

  ```
  $ oc new-project cicd --display-name="CI/CD"
  ```

Create the CI/CD compoentns based on the provided template

  ```
  $ oc process -f cicd-gogs-template.yaml | oc create -f -
  ```

Create Dev and Stage projects for Tasks JAX-RS application

  ```
  $ oc new-project dev --display-name="Tasks - Dev"
  $ oc new-project stage --display-name="Tasks - Stage"
  ```

Jenkins needs to access OpenShift API to discover slave images as well accessing container images. Grant Jenkins service account enough privileges to invoke OpenShift API for the created projects:

  ```
  $ oc policy add-role-to-user edit system:serviceaccount:cicd:default -n cicd
  $ oc policy add-role-to-user edit system:serviceaccount:cicd:default -n dev
  $ oc policy add-role-to-user edit system:serviceaccount:cicd:default -n stage
  ```
Go on ```Jenkins administration > Plugins configuration > Advanced``` and set proxy settings (```proxy-internet.internal.fr:3128```). Save and check for plugin updates. Then, go on plugin updates (```Jenkins administration > Plugins configuration```) and update only the Kubernetes plugin. Don't forget to restart Jenkins.

Then, before launch the pre-installed pipeline, you have to open it and save it once (without doing any modification) otherwise it will fail.

# Things to know

On our infrastructure, the main issue is with proxies. The real advantage of using Jenkins on OpenShift infrastrucutre is to dedicate one pod to one action (like executing a Jenkins pipeline). But, to do that, pods must communicate between us and our proxy configuration prevent them to do that correctly. All the modifications I did was to configure the pods to do not use proxy, but at each step (barely), you have to do that at a different level.

Known issue : The Docker image of the slave pod not updating automatically when I push some modifications on GitHub, so sometimes the slave pod is built on an older image than the latest (and without last modifications/fixes). Temporarilty workaround : Manually delete docker images of slave pod (```sudo docker rmi```) on each node and rebuild the jdk-jenkins-slave image.

The last thing I worked on and which is not fixed now : At one moment during the pipeline execution, the slave pod will try to get a .pom file of jBoss on our nexus local installation but by default, the request goes through proxy so it fails. We have two ways to fix that :
* Configure to no proxy on jenkins-slave/settings.xml (best solution)
* Expose our nexus on internet


# Demo Guide

1. RunJenkins has the Pipeline plugin pre-installed. A Jenkins pipeline job is also pre-configured which clones Tasks JAX-RS application source code from GitHub, builds, deploys and promotes the result through the deployment pipeline. Click on ```tasks-cd-pipeline``` and _Configure_ and explore the pipeline definition.

2. If using Gogs, modify the git repository url in the pipeline definition and set it to ```http://gogs:3000/gogs/openshift-tasks.git```.

2. Run an instance of the pipeline by starting the ```tasks-cd-pipeline``` job.

2. During pipeline execution, verify a new Jenkins slave pod is created withing _CI/CD_ project to execute the pipeline.

3. After pipeline completion, demonstrate the following:
  * Explore the ```snapshots``` repository in Nexus and verify ```openshift-tasks``` is pushed to the repository
  * Explore SonarQube and verify a project is created with metrics, stats, code coverage, etc
  * Explore _Tasks - Dev_ project in OpenShift console and verify the application is deployed in the DEV environment
  * Explore _Tasks - Stage_ project in OpenShift console and verify the application is deployed in the STAGE environment  

4. Add a webhook in [GitHub](https://developer.github.com/webhooks/creating/#setting-up-a-webhook) or [Gogs](https://gogs.io/docs/features/webhook) to trigger the pipeline whenever a change is pushed to the git repository. Use pipeline job's _Build Now_ url as the webhook url.

  If using Gogs, webhooks configuration is in repository's _Settings &gt; Webhooks_ and the _tasks-cd-pipeline_ webhook url is http://jenkins:8080/job/tasks-cd-pipeline/build?delay=0sec.

  _Note:_ if GitHub is used and Jenkins route is not accessible from the Internet, use SCM Polling instead of webhooks to trigger builds.

5. Clone the ```openshift-tasks``` git repository and using an IDE (e.g. JBoss Developer Studio), remove the ```@Ignore``` annotation from ```src/test/java/org/jboss/as/quickstarts/tasksrs/service/UserResourceTest.java``` test methods to enable the unit tests. Commit and push to the git repo.

6. Check out Jenkins, a pipeline instance is created and is being executed. The pipeline will fail during unit tests due to the enabled unit test.

7. Check out the failed unit and test ```src/test/java/org/jboss/as/quickstarts/tasksrs/service/UserResourceTest.java``` and run it in the IDE.

8. Fix the test by modifying ```src/main/java/org/jboss/as/quickstarts/tasksrs/service/UserResource.java``` and uncommenting the sort function in ```getUsers``` method.

9. Run the unit test in the IDE. The unit test runs green. Commit and push the fix to the git repository and verify a pipeline instance is created in Jenkins and executes successfully.

![](https://github.com/OpenShiftDemos/openshift-cd-demo/blob/openshift-3.2/images/jenkins-pipeline.png)
