# CI/CD with Cloudbees on VKE Workshop

This workshop is presented by VMware and Cloudbees

Instructions created by Dan Illson and Jeff Fry

Presented by Valentina Alaria, Jeff Fry, Dan Illson, Sean O'Dell, and Bill Shetti

## Prerequisites

In order to successfully complete this workshop, you will need:

1. A terminal shell (preferably bash) or emulator
2. [A github account](https://github.com/)
3. [A Dockerhub account](https://hub.docker.com/)
4. The [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) command line package configured within the shell 
5. The kubectl utility installed (this can be installed from the VKE UI)
6. [Helm](https://docs.helm.sh/using_helm/#installing-helm) installed
7. A login for the VMware Kubernetes Engine service (provided at check-in)

If you need to install any of the prerequsites or signup for a github account, this can be done while the VMware Kubernetes Engine cluster is being built or the Cloudbees core components/Jenkins master are being provisioned.

Please follow the links above for instructions to install or request the prerequisite items.

## Provision a VKE cluster

### Login to VKE UI and Download the CLI package

Download the CLI package from the button in the bottom left corner of the screen labeled 'Download CLI' and select the correct operating system. Make sure it has execute permissions.

### Login to the VKE CLI

Use the following format, replacing organization-id with your organization ID and refresh-token with your refresh token.
```yaml
vke account login -t organization-id -r refresh-token
```

The organization ID is available in the VKE UI by clicking on the box showing your username and organization name in the upper right portion of the screen. Please click on the Org ID to get the long form and use that long form in the command.

For an API/refresh token, please click again on the box displaying your username and organization name. Then please click on the 'My Account' button.

On the resulting page, please select 'API tokens' the third option in the horizontal navigation bar under 'My Account'. If you have an existing token, please copy and use it, otherwise please click the button labelled 'New Token' and then copy the result.

Copy these values down to a text file, they will be important during Jenkins configuration later.

### Create a cluster

To create a VKE cluster, run the following command:
```yaml
vke cluster create --name <cluster name> --region us-west-2 -f sharedfolder -pr sharedproject -v 1.10.2-59 --privilegedMode
```

For region please use the value 'us-west-2'

You will be asked to acknowledge the creation of a piviliged mode Kubernetes cluster. Please enter 'Y' at the prompt.

### Get Kubectl and Helm

If it isn't already installed on your machine, please install the kubectl command line package for managing Kubernetes.

Kubectl can be pulled down in a variety of ways, including from the VKE UI (on the page for a smartcluster, select actions and then select the correct operating system for your device).

For Helm, please see the following page: [Helm](https://github.com/helm/helm)
For installing Helm on an Ubuntu system, you can simply use 
```
sudo snap install helm --classic
```

### Access the VKE Cluster

To gain kubectl access to your VKE cluster via the command line use the following command:
```yaml
vke cluster merge-kubectl-auth <cluster name>
```

This will authenticate kubectl to your VKE cluster and set the correct context to access that cluster.

Correct funtionality can be verfied by successfully running 'kubectl get' commands against the cluster (kubectl get nodes, kubectl get pods, etc)

### Initiailize Helm

To setup Tiller (the cluster side component of Helm) in your VKE cluster, run the following command:
```yaml
helm init
helm repo update
```

The 'helm repo update' command pulls the most recent version of the charts in any repositiories mapped into Helm (stable by default) so as to avoid installing older versions of these components, which might introduce issues.

The `helm init` command will take a couple of minutes to run on the VKE cluster as it will cause the cluster to scale up and add a worker node.
This is expected, please move on to the 

## Clone this repository to your local machine and create an online copy

In order to feed the CI/CD pipeline automatically on a git push, you'll need to create an individual copy of this repository.

### Fork this repository

In order to have your own working copy of this repository, you'll want to fork it into your account.
1. First navigate to [http://github.com] and login to your account
2. Then navigate to the page for this repository [https://github.com/dillson/jw-workshop]
3. In the upper-right hand section of the screen, click on the button labeled 'Fork'

A copy of this repository will then be forked into your account

### Clone the repository

First pick a location on your local machine to clone the forked repository.

After ensuring that 'git' is installed, run these commands:
```
cd jw-workshop
git clone https://github.com/<your username>/jw-workshop.git
```
Comgratulations, you know have a local copy of this repository. It will be in a folder labeled 'jw-workshop'

### Edit the Jenkinsfile

From the cloned repo root folder, run the following commands:
```
sed -i 's/bbrundert/<your dockerhub username not email>/g' Jenkinsfile
sed -i 's/bbrundert-devops/<your VKE cluster name>/g' Jenkinsfile
sed -i 's/bbrundert/<your dockerhub username not email>/g' deployFiles/deployment.yaml
```
(please make sure the deployFiles/deployment.yaml refers to your container image on DockerHub, otherwise you won't see changes after updating the applications later in the workshop)

The cluster name is whatever you named your VKE cluster during the VKE cluster provisioning steps previously.

Now run:
```
git add Jenkinsfile
git commit -m 'altered Jenkinsfile for my parameters'
git push -u origin master
```

These commands add, commit, and push these changes to your online repository for the pipeline to read later.

**Stay in this folder for the next steps**

## Helm Chart for CloudBees Core on VMware Kubernetes Engine (VKE) - based on [This repo by Jeff Fry](https://github.com/cloudbees/core-helm-vke)

### Create the Helm Chart

From folder 'jw-workshop' (root of the cloned repository), run:
```
helm package core-helm-vke/CloudBeesCore
```

### Installation Instructions

1. Install Helm if not already installed.
```
helm init
```

2. Wait for the tiller pod to come up. This may take a few minutes.
```
kubectl -n kube-system get pods
```
```
NAME                             READY     STATUS    RESTARTS   AGE
tiller-deploy-5c688d5f9b-l27kk   1/1       Running   0          3m
```
3. Create a namespace for CloudBees Core to be installed in. 

```
kubectl create namespace cloudbees
```

4. Install an Nginx Ingress Controller with Helm. Ensure you specify the controller scope using the cloudbees namespace
```
kubectl create namespace ingress-nginx
```
```
kubectl create clusterrolebinding nginx-ingress-cluster-rule --clusterrole=cluster-admin --serviceaccount=ingress-nginx:nginx-ingress
```
```
helm install --namespace ingress-nginx --name nginx-ingress stable/nginx-ingress --version 0.23.0 --set rbac.create=true --set controller.service.externalTrafficPolicy=Local --set controller.scope.enabled=true --set controller.scope.namespace=cloudbees
```
4. Wait for the _Load Balancer Ingress_ field to populate. This is the hostname that will be used in the next step. This may take a few minutes and require multiple runs before the value populates.
```
kubectl describe service nginx-ingress-controller -n ingress-nginx
```
5. Install CloudBees Core. The <lb-ingress-hostname> value is included in the output of the previous command. 
(It looks like nginx-ingress-controller.<cluster-name>-<random>.vke-user.com)

```
helm install core-helm-vke/cloudbeescore --set cjocHost=<lb-ingress-hostname> --namespace cloudbees
```
 
If that does not work, try with deploying the local Helm Chart that was created earlier (see https://github.com/dillson/jw-workshop/issues/2):
```
helm install CloudBeesCore-0.2.0.tgz --set cjocHost=<lb-ingress-hostname> --namespace cloudbees
```
 
6. Monitor the progress.
```
kubectl rollout status sts cjoc --namespace cloudbees
```
7. Wait for success message.
```
statefulset rolling update complete 1 pods at revision cjoc-59cc694b8b...
```
8. Go to ```http://<lb-ingress-hostname>/cjoc```
9. Get the initial admin password.
```
kubectl exec cjoc-0 cat /var/jenkins_home/secrets/initialAdminPassword --namespace cloudbees
```
10. Follow the instructions in the setup wizard. Request a trial license. Select the 'Install selected plugins' option. 
  * If prompted for an 'Operations Center Upgrade', please ignore and continue without it.
  * If you have issues requesting the trial license (or later on when working with Jenkins), try to restart the Pod (see https://github.com/dillson/jw-workshop/issues/3):
```
jw-workshop$ kubectl delete pod cjoc-0 --namespace=cloudbees
```

## Cloudbees configuration and pipeline creation

### Team Onboarding
1. Click on the _Teams_ menu item.
2. Follow the team creation wizard.
3. Specify a name for team.
4. Choose an icon.
5. Select the 'Basic' team recipe.
6. Wait for a few minutes for the Jenkins Master to be created.
7. Use the button just to the left of the 'Logout' button that looks like an exit sign to reach the classic Jenkins UI. The alt-text for this button is 'Go to classic.' It takes you from the modern UI back to the more detailed classic UI. 

### Jenkins Plugin Configuration

1. Use the horizontal navigation bar at the top of the screen to navigate to the leftmost 'Jenkins'
2. From the vertical navigation bar on the left edge of the screen, select 'Manage Jenkins' -> 'Manage Plugins'
3. Select the 'Available' tab from the top of the main panel. Then search down the list for 'GitHub plugin'. Check the box for this plugin, then click the 'Install without restart' button at the bottom of the screen

### Configure credentials

Return to the main screen of the Cloudbees Jenkins Operations Center by using the horizontal navigation bar at the top of the screen again. Click on the leftmost entry 'Jenkins'

1. Click on the 'Teams >> {name}' link in the middle of the main panel. The name will be the value you specified when creating a team earlier.
2. Select 'Credentials' from the vertical navigation bar on the left edge of the screen
3. Look for the section labeled 'Stores scoped to Jenkins'
4. Under that section, click on the link for the (global) domain
5. From the vertical nav bar on the left edge of the screen, click 'Add Credentials'
6. Select 'Username with password' from the 'Kind' dropdown menu at the top of the screen.
7. Leave 'Scope' set to Global
8. In the Username field, enter your dockerhub user name (not email!!). In Password, enter your dockerhub password. **For ID and description, you must use 'dockerhub'**
9. Click 'OK'
10. Click 'Add Credentials' from the vertical navigation bar again.
11. Use 'Username with password' from the 'Kind' dropdown menu. Use 'Global' as the scope.
12. For username, use your VKE orgnaization ID from the cluster provisioning stage.
13. For password, use your VKE API/refresh token noted down during VKE cli login.
14. **For ID, you must use 'VCS'.** Add a relevant description for your reference.
15. Click 'OK'

### Pipeline setup

Return to the main screen of the Cloudbees Jenkins Operations Center by using the horizontal navigation bar at the top of the screen again. Click on the leftmost entry 'Jenkins'

1. Click on the 'Teams >> {name}' link in the middle of the main panel. The name will be the value you specified when creating a team earlier.
2. From the vertical navigation bar on the left edge of the screen, click 'New Item'
3. Select the 'Pipeline' type from the list. Be sure to give it a name. Then click the 'OK' button at the bottom of the screen.
4. On the following screen, Check the box in the 'General' section reading 'GitHub Project'. A text box will then appear and ask for the project URL. Use the URL of **your** forked Github repository. Likely of the form, "github.com/'username'/jw-workshop".
5. Under the 'Build Triggers' section, check the box labeled 'GitHub hook trigger for GITScm polling'. This will enable the desired 'build on git push' behavior.
6. In the 'Pipeline' section, location the 'Definition' dropdown menu. In that dropdown, select 'Pipeline script from SCM'
7. From the 'SCM' sub dropdown menu, select 'Git'
8. In the 'Repositor URL' text field, enter the URL of your forked repository. 
9. Ensure the 'Credentials' dropdown menu value is '- none -'.
10. Locate the 'Script Path' text field menu and enter 'Jenkinsfile'
11. Click 'Apply', the 'Save' at the bottom of the screen.

### Execute the pipeline once to validate functionality

From the Pipeline status screen (reached by navigating to your pipeline and using the vertical nav bar at the left).
Once here, click the 'Build Now' link in the vertical nav bar on the left edge of the screen to execute the pipeline once on-demand.

### Access the app to validate

1. On your local machine, run : `vke cluster show <cluster name>`.
2. Look for the 'Address:' line, then copy the URL displayed on that line.
3. In your browser, navigate to `http://<address value>:30400`
4. You should be on the root web page of an express (node.js) app welcoming you to the workshop. There are also 2 subpages:
  * /vke
  * /cloudbees

## Extended activities

This section represents extended activities for those who complete the base workshop. The section fully automates the CI/CD pipeline to trigger off of a push event to the git repository.

### Configure Github Webook

Return to the main screen of the Cloudbees Jenkins Operations Center by using the horizontal navigation bar at the top of the screen again. Click on the leftmost entry 'Jenkins'

1. From the vertical navigation bar on the left edge of the screen, select 'Manage Jenkins' -> 'Configure System'
2. Scroll down until you come to the 'Github' section of the conifg
3. Click the button for 'Add GitHub Server'
4. The API URL text field should read, `https://api.github.com`. If not, please edit it to that value
5. Ensure that the 'Manage Hooks' box is checked.
6. Locate the bottommost 'Advanced' button in the GitHub section.
7. In the 'Additional Actions' sub section, click the 'Manage additional GitHub actions' dropdown and the 'Convert login and password to token' option
8. Select the radio button for 'From login and password'
9. Enter your Github username and password, then click the 'Create token credentials' button.
10. Click the 'Apply' button at the bottom of the screen.
11. Under the 'API URL' text field at the top of the GitHub section, there is a credentials dropdown. Select the credential you just created. It will start with `Github (https://api.github.com) auto generated token credentials`.
12. Click the 'Apply' button at the bottom of the screen.
13. In the 'Shared Secret' dropdown menu, select the same option as 'Credentials' dropdown above.
14. Click the 'Apply' button at the bottom of the screen. Then click the 'Save' button.

### Test by altering the code and pushing the changes

1. Alter the app code. Pick from the set of `*.hbs` files in the views/ folder of the repository.
2. Save the changes to the HTML body files
3. Run the `git add .` command to add all files to git
4. Run `git commit -m 'testing pipeline trigger'` to commit the changes
5. Run `git push -u origin master` to upload the code to the repo
6. Go the the pipeline status screen for your pipeline to verify that the pipeline has triggered
7. Verify proper application functionality
