---
layout: post
title: Add Git Hooks in Business Central
date: 2020-06-22
tags: [ jBPM ]
---

As we introduced the jBPM architecture in [this post], it mainly consists in two components:
- Business/Decision central as designer/management tools of our business processes
- Kie Server as execution servers

In this post, we'll be focusing on Business Central deployed on Openshift. The main use case of this component is to create/manage projects where each project is composed by a set of business logic assets. 

Why is important to setup a [Git Hook](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) in Business Central?

Because each project in Business Central is kept in a GIT repository within the Business Central component. Everytime we update the project (adding new business logic assets or updating them), the changes will be pushed in the internal GIT repository only. What about if we want to have the sources in a private repository on Github? As private repositories require SSH, we need to configure Git hooks to push the changes and the SSH files to get authorization.

| Business/Decision central only works for post-commits, not pre-commits!

## 0. Prepare the environment

We need to have:
- An Openshift instance and project
- [Openshift CLI](https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html)
- [Business Automation operator](https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.3/html/deploying_a_red_hat_process_automation_manager_environment_on_red_hat_openshift_container_platform_using_operators/operator-con) deployed in the Openshift project

And make sure we're in the correct project:

```
oc project my-project
```

## 1. Create your post-commit script

Business/Decision central only works for post-commits, not pre-commits. So, let's create the *post-commit* script:

```
#!/usr/bin/sh
echo "running post-commit git hook"
git remote add upstream git@github.com:Sgitario/my-private-repository.git
git push upstream
```

## 2. Upload the file into Openshift

```
oc create configmap githook-post-commit --from-file=post-commit=post-commit
```

## 3. Create the secrets for SSH access

```
oc create secret generic githook-ssh-key-secret --from-file=id_rsa=/path/to/.ssh/id_rsa --from-file=known_hosts=/path/to/.ssh/known_hosts
```

Where */path/to* is the path where the git *id_rsa* and *known_hosts* files are. 

## 4. Create the KieApp instance

```yaml
apiVersion: app.kiegroup.org/v2
kind: KieApp
metadata:
  name: myapp
spec:
  commonConfig:
    adminPassword: admin # Business Central password
    adminUser: admin # Business Central username
    applicationName: myapp
  environment: rhpam-trial # This is important: Trial must be used only for dev purposes!
  objects:
    console: # Business Central 
      gitHooks: 
        from:
          kind: ConfigMap
          name: githook-post-commit
        sshSecret: githook-ssh-key-secret
  version: 7.8.0 # This is also important! As the "sshSecret" is new from 7.8.0
```

The **Business Automation** operator will watch this KieApp and deploy all the required components. 

## 5. Test your project

When we create a new project using the Business Automation site, we should see the push in the server logs.

## Appendix: Another approach using Custom Images

As the above method only works from 7.8.0 images, if we're using older versions, we can create a custom image to achieve the same. This method requires also a [Quay](https://quay.io) account.

```
docker login quay.io
```

And provide your credentials.

1. Create a Dockerfile which will add the git hook to the /opt/kie/data/git/hooks/ directory of the Business Central

In this example, we're using the Business Central 7.7.1 version. 

*Dockerfile:*
```
FROM registry-proxy.engineering.redhat.com/rh-osbs/rhpam-7-rhpam-businesscentral-rhel8:7.7.1-1
COPY post-commit /opt/kie/data/git/hooks/
COPY /path/to/.ssh /home/jboss/.ssh/
```

| The *post-commit* file is the same one as above.

2. Build the new Business Central image which will contain the git hook:

```
podman build . --tag quay.io/<my-namespace>/rhpam-7-rhpam-businesscentral-with_githooks
```

3. Verify the git hook is in the custom Business Central image:

```
podman run -it --rm quay.io/<my-namespace>/rhpam-7-rhpam-businesscentral-with_githooks /bin/bash
> cat /opt/kie/data/git/hooks/post-commit
```

4. Push the extended image to your registry:

```
podman push quay.io/<my-namespace>/rhpam-7-rhpam-businesscentral-with_githooks
```

5. Deploy your application and follow instructions below:

https://access.redhat.com/documentation/en-us/red_hat_process_automation_manager/7.7/html/deploying_a_red_hat_process_automation_manager_environment_on_red_hat_openshift_container_platform_using_operators/operator-con#operator-deploy-central-proc

Finally, you need to manually update the ImageStream that Business Central is using so the Docker image points out to your Quay custom business central.