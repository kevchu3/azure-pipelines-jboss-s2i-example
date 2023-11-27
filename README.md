# Azure Pipelines JBoss S2I Example

This is a reference application pipeline using Azure Pipelines to build a JBoss application using Source-to-Image (S2I).

## This Repository

This repository uses the `helloworld` example from jboss-eap-quickstarts GitHub repository which is placed in the [src](src) directory:
https://github.com/jboss-developer/jboss-eap-quickstarts/tree/7.4.x

The [Dockerfile](Dockerfile) uses instructions from Red Hat Software Collections 3, [Building Application Images from Dockerfiles Using S2I Scripts](https://access.redhat.com/documentation/en-us/red_hat_software_collections/3/html/using_red_hat_software_collections_container_images/building_application_images#building_application_images_from_dockerfiles_using_s2i_scripts)

## Staging the Builder Image in ACR

Push Red Hat image to Azure Container Registry.  More documentation on using [ACR](https://learn.microsoft.com/en-us/azure/container-registry/).  For this example, we'll use `<acr-registry-name>.azurecr.io` which you can replace with your own.

```
$ az login
$ az acr login --name <acr-registry-name>.azurecr.io
You can perform manual login using the provided access token below, for example: 'docker login loginServer -u 00000000-0000-0000-0000-000000000000 -p accessToken'
...
The resulting output will provide an access token
...
$ podman login <acr-registry-name>.azurecr.io -u 00000000-0000-0000-0000-000000000000 -p accessToken
```

We will use the [JBoss Web Server builder image](https://catalog.redhat.com/software/containers/jboss-webserver-5/jws57-openjdk17-openshift-rhel8/6241a7c7456d2e626dd773bd)
```
$ podman login registry.redhat.io
$ podman pull registry.redhat.io/jboss-webserver-5/jws57-openjdk17-openshift-rhel8:latest
$ podman tag registry.redhat.io/jboss-webserver-5/jws57-openjdk17-openshift-rhel8 <acr-registry-name>.azurecr.io/jboss-webserver-5/jws57-openjdk17-openshift-rhel8
$ podman push <acr-registry-name>.azurecr.io/jboss-webserver-5/jws57-openjdk17-openshift-rhel8 --remove-signatures
```

## Building the Image

Import this Git repository into Azure Pipelines, and the trigger will automatically begin to build the image.  Note that this pipeline uses an agent pool named Default.

## Deploying the Image

We'll create an OpenShift secret for ACR, a standalone pod (not best practices), and link the secret to the default serviceaccount.  We'll then rsh into the pod and confirm that it's printing Hello World!

```
$ oc new-project helloworld
$ oc create secret docker-registry acr --docker-username 00000000-0000-0000-0000-000000000000 --docker-server <acr-registry-name>.azurecr.io --docker-password <password>
$ oc secrets link default acr --for=pull
$ oc run helloworld --image=<acr-registry-name>.azurecr.io/my-dev/helloworld-java:<tag>
$ oc rsh helloworld 
sh-4.4$ curl localhost:8080
Hello World!
```

