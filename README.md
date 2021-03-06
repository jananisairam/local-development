**NOTE:** If you have already setup local development before version ``v0.11.0``, please delete your minikube instance and start afresh. You can take dump of the database if you want to keep the data.

# Pre-requisites

- Install [minikube](https://github.com/kubernetes/minikube/releases) (version ``0.18.0``) . Do not install version ``0.19.0``. There are issues with kube-dns shipped with minikube 0.19.

- Install latest kubectl (>= 1.6.0) (https://kubernetes.io/docs/tasks/kubectl/install/)

# Instructions:

- Start minikube:

  ```
  minikube start --kubernetes-version v1.6.3
  ```

- Clone this repo and cd into the directory.
  ```
  git clone git@github.com:hasura/local-development.git
  cd local-development
  ```

- Edit ``controller-configmap.yaml`` and set the ``gatewayIp`` field to the ip of your minikube instance. You can get this by running:

  ```
  minikube ip
  ```

- Now create the necessary resources to setup the platform
  ```
  kubectl create -f project-conf.yaml
  kubectl create -f project-secrets.yaml
  kubectl create ns hasura
  kubectl create -f controller-configmap.yaml
  ```

**NOTE:** The next steps will roughly download about 1-1.5GB of docker images.

- Run ``kubectl create -f platform-init.yaml``. This will initialise the state for the platform. It can take a while, follow the logs by running:

  ```
  kubectl logs platform-init-v0.11.0 -n hasura --follow
  ```

  When you see a line that says "successfully initialised the platform's state", you can move onto the next step.

- Now, run ``kubectl create -f platform-sync.yaml``. This will bring up all the services and keeps them in sync with the project conf. You can either follow the logs of the platform-sync pod in the hasura namespace or check the status of the project as follows:

  ```
  kubectl get cm hasura-project-status -o json --watch
  ```

  This outputs some JSON. We wait for the ``services`` key's summary to say ``Synced``. When it does, the platform is up and running.

- To access the platform on a domain, we make use of ``vcap.me``, a domain which always points to 127.0.0.1. For this to work, we have to setup port forwarding:
  - For Windows, follow instructions here: https://github.com/hasura/support/issues/250#issuecomment-299862303.
  - On Linux/Mac, Run:

    ```
    export GW_IP=$(minikube ip) && sudo ssh -L 80:$GW_IP:80 -L 2022:$GW_IP:2022 docker@$GW_IP
    ```
    The default password for `docker` user is `tcuser`. This will forward port 80 (and hence the sudo) and 2022 on your local machine to the minikube cluster.
- Your Hasura project should now be accessible at http://console.vcap.me.
  ``kubectl -n hasura get pods`` shows all the Hasura platform services as running (data, auth, console, sshd, postgres, session-redis etc.)
- Login to the console with: ``admin``, ``adminpassword``
- Postgres login: ``admin``, ``pgpassword``

# Errors:

When ``platform-init`` fails, we have to clean up the state as follows before trying again

  ```
  # Clean up the state
  kubectl delete configmap hasura-project-status
  kubectl delete ns hasura # This will take some time
  minikube ssh "sudo rm -rf /data/hasura.io"
  kubectl create ns hasura

  # Run init once again
  kubectl create -f platform-init.yaml
  ```

# Cleanup/Uninstall the project

This will cleanup/uninstall the project including the data in the database.

- Delete the created resources:

  ```
  kubectl delete ns hasura
  kubectl delete configmap hasura-project-status
  kubectl delete configmap hasura-project-conf
  kubectl delete secret hasura-project-secrets
  ```
- Delete the data directory:

  **Be careful!**. This will delete all data in the database. Take a dump of the data before this if you want to.

  ```
  minikube ssh "sudo rm -rf /data/hasura.io"
  ```
If you want to install again, you can start from the "Instructions" sections in this README.

# Exposing local minikube to public Internet

Follow the instructions here: https://github.com/hasura/ngrok

# Upcoming features
- Migration from local development (minikube) to Kubernetes cluster on any cloud provider.
