# compose-to-skaffold-tutorial

## Intro

Visit the following blog post for more context:
https://testingclouds.wordpress.com/2021/03/09/migrating-from-docker-compose-to-skaffold/

This tutorial will walk you through the same process of moving an app from a Docker Compose development setup to using [Skaffold](https://skaffold.dev) and [Minikube](https://minikube.sigs.k8s.io/docs/). In this case, the app is Taiga which is an open source "project management tool for multi-functional agile teams".

If you have a Google account and want to try it in a free sandbox environment click here:

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.svg)](https://ssh.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2Fviglesiasce%2Fcompose-to-skaffold-tutorial&cloudshell_workspace=.&cloudshell_tutorial=README.md)

![image](https://user-images.githubusercontent.com/410279/110428648-7a2d4380-805e-11eb-8744-8ded85136b5d.png)

## Tutorial

1. Start minikube.

    ```sh
    minikube start
    ```

1. Clone the docker-taiga repository which contains the code necessary to get Taiga up and running with Docker Compose.
 
    ```sh
    git clone https://github.com/docker-taiga/taiga
    cd taiga
    ```

1. The Docker Compose file in docker-taiga doesn’t set ports for all the services which is required for proper discoverability when things are converted to Kubernetes. Apply the following patch to ensure each service exposes its ports properly.

    ```shell
    cat > compose.diff <<EOF
    diff --git a/docker-compose.yml b/docker-compose.yml
    index e09d717..94920c8 100644
    --- a/docker-compose.yml
    +++ b/docker-compose.yml
    @@ -12,0 +13,2 @@ services:
    +    ports:
    +      - 80:8000
    @@ -22,0 +25,2 @@ services:
    +    ports:
    +      - 80:80
    @@ -33,0 +38,2 @@ services:
    +    ports:
    +      - 80:80
    @@ -46,0 +53,2 @@ services:
    +    ports:
    +      - 5432:5432
    @@ -55,0 +64,2 @@ services:
    +    ports:
    +    - 5672
    diff --git a/variables.env b/variables.env
    index 4e48c17..8de5b09 100644
    --- a/variables.env
    +++ b/variables.env
    @@ -1 +1 @@
    -TAIGA_HOST=taiga.lan
    +TAIGA_HOST=localhost
    @@ -3 +3 @@ TAIGA_SCHEME=http
    -TAIGA_PORT=80
    +TAIGA_PORT=4506
    EOF
    patch -p1 < compose.diff
    ```

1. Next, we’ll bring in the source code for one of Taigas components that we want to develop on, in this case the backend. We’ll also set our development branch to the 6.0.1 tag that the Compose File was written for.

    ```sh
    git clone https://github.com/taigaio/taiga-back -b 6.0.1
    ```

1. Now that we have our source code, Dockerfile, and Compose File, we’ll run skaffold init to generate the skaffold.yaml and Kubernetes manifests.

    ```sh
    skaffold init --compose-file docker-compose.yml
    ```
  
    Skaffold will ask which Dockerfiles map to the images in the Kubernetes manifests.
    
    **The default answers are correct for all questions except the last. Make sure to answer yes (y) when it asks if you want to write out the file.**
  
        ? Choose the builder to build image dockertaiga/back Docker (taiga-back/docker/Dockerfile)
        ? Choose the builder to build image dockertaiga/events None (image not built from these sources)
        ? Choose the builder to build image dockertaiga/front None (image not built from these sources)
        ? Choose the builder to build image dockertaiga/proxy None (image not built from these sources)
        ? Choose the builder to build image dockertaiga/rabbit None (image not built from these sources)
        ? Choose the builder to build image postgres None (image not built from these sources)
        apiVersion: skaffold/v2beta12
        kind: Config
        metadata:
        name: taiga
        build:
        artifacts:
        - image: dockertaiga/back
          context: taiga-back/docker
          docker:
            dockerfile: Dockerfile
        deploy:
        kubectl:
          manifests:
          - kubernetes/back-claim0-persistentvolumeclaim.yaml
          - kubernetes/back-claim1-persistentvolumeclaim.yaml
          - kubernetes/back-deployment.yaml
          - kubernetes/db-claim0-persistentvolumeclaim.yaml
          - kubernetes/db-deployment.yaml
          - kubernetes/default-networkpolicy.yaml
          - kubernetes/events-deployment.yaml
          - kubernetes/front-claim0-persistentvolumeclaim.yaml
          - kubernetes/front-deployment.yaml
          - kubernetes/proxy-claim0-persistentvolumeclaim.yaml
          - kubernetes/proxy-deployment.yaml
          - kubernetes/proxy-service.yaml
          - kubernetes/rabbit-deployment.yaml
          - kubernetes/variables-env-configmap.yaml

        ? Do you want to write this configuration to skaffold.yaml? Yes

    We’ll also fix an issue with the Docker context configuration that Skaffold interpreted from the Compose File. The taiga-back repo keeps its Dockerfile in a sub-folder rather than the top-level and uses the context from the root. Also

    ```sh
    sed -i 's/context:.*/context: taiga-back/' skaffold.yaml
    sed -i 's/dockerfile:.*/dockerfile: docker\/Dockerfile/' skaffold.yaml
    sed -i 's/name: TAIGA_SECRET/name: TAIGA_SECRET_KEY/' back-deployment.yaml
    ```

1. Now we’re ready to run Skaffold’s dev loop to continuously re-build and re-deploy our app as we make changes to the source code. Skaffold will also display the logs of the app and even port-forward important ports to your machine.

    You should start to see the backend running the database migrations necessary to start the app and be able to get to the web app via [this link]( http://localhost:4056).

    ```sh
    skaffold dev --port-forward
    ```
  
1. To initialize the first user, run the following command in a new terminal and then log in to Taiga with the username admin and password 123123.

    ```sh
    kubectl exec deployment/back python manage.py loaddata initial_user
    ```

Congrats! You’ve now transitioned your development tooling from Docker Compose to Skaffold and Minikube.

![image](https://user-images.githubusercontent.com/410279/110429760-3fc4a600-8060-11eb-8e25-bc2faa702c42.png)
