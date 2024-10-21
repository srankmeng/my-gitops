# my-gitops

## Running JSON Server Locally

To run JSON Server locally, follow these steps:

1. Ensure you have docker & docker compose installed on your machine.

2. Start JSON Server with docker compose:

    ```sh
    docker compose -f json-server/docker-compose.yml up
    ```

    This will start the server on `http://localhost:3000`.

3. Available Routes

    Based on the `db.json` file, the following routes are available:

    ```sh
    - GET    /posts
    - GET    /posts/:id
    - POST   /posts
    - PUT    /posts/:id
    - PATCH  /posts/:id
    - DELETE /posts/:id

    - GET    /comments
    - GET    /comments/:id
    - POST   /comments
    - PUT    /comments/:id
    - PATCH  /comments/:id
    - DELETE /comments/:id

    - GET    /profile
    ```

## Running JSON Server on Local Kubernetes with k3d

Note: Make sure your Docker daemon is running before creating the k3d cluster.

To run JSON Server on a local Kubernetes cluster using k3d, follow these steps:

1. Install k3d:
    Follow the installation instructions for your operating system from the [k3d documentation](https://k3d.io/#installation).

2. Create a local Kubernetes cluster:

    ```sh
    k3d cluster create my-cluster --servers 1 --agents 1 --port "8888:80@loadbalancer" --port "8889:443@loadbalancer"
    ```

3. Import json-server image to local Kubernetes cluster:

    ```sh
    k3d image import my_json_server:1.0 --cluster my-cluster
    ```

4. Apply the Kubernetes manifests:

    ```sh
    kubectl apply -f k8s/service.yaml
    kubectl apply -f k8s/ingress.yaml
    ```

5. The service to access it locally:

    Now you can access the JSON Server at `http://localhost:8888`.

6. Clear json-server resources the Kubernetes manifests:

    ```sh
    kubectl delete -f k8s/service.yaml
    kubectl delete -f k8s/ingress.yaml
    ```

## GitOps with ArgoCD on k3d Cluster

To implement GitOps using ArgoCD on your k3d cluster, follow these steps:

1. Install ArgoCD on your k3d cluster:

    ```sh
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

2. Wait for ArgoCD pods to be ready:

    ```sh
    kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
    ```

3. Access the ArgoCD UI:

    Port-forward the ArgoCD server:

      ```sh
      kubectl port-forward svc/argocd-server -n argocd 8080:443
      ```

    Then, access the ArgoCD UI at `https://localhost:8080`

4. Login username as `admin` and retrieve the initial admin password:

    ```sh
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo
    ```

5. Create an Application in ArgoCD:

    Create a file named `argocd/application.yaml` with the following content:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: json-server-app # application name
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: 'https://github.com/srankmeng/my-gitops.git'
        path: k8s # directory keep k8s manifest files
        targetRevision: HEAD
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: default
      syncPolicy:
        automated:
          prune: true # auto delete resource when haven't on git repository
          selfHeal: true # auto sync state compare on git repository
    ```

6. Apply the Application:

    ```sh
    kubectl apply -f argocd/application.yaml
    ```

7. ArgoCD will now automatically sync your Kubernetes manifests from the specified Git repository to your k3d cluster. Any changes pushed to the repository will be automatically applied to the cluster.

8. Recheck application on ArgoCD UI:

    Access the ArgoCD UI at `http://localhost:8080`.

9. The service to access it locally:

    Now you can access the JSON Server at `http://localhost:8888`.

10. To make changes to your application:
    - Update the Kubernetes manifests in your Git repository: for example replica
    - Commit and push the changes
    - ArgoCD will detect the changes and automatically apply them to your k3d cluster

This setup ensures that your k3d cluster configuration is always in sync with your Git repository, following GitOps best practices.

## Running any application with Helm Chart

Deploy Grafana using ArgoCD with Helm Chart:

1. Create a new file named `argocd/grafana-application.yaml` with the following content:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: grafana
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: 'https://grafana.github.io/helm-charts'
        chart: grafana
        targetRevision: 6.50.7  # Use the latest stable version
        helm:
          values: |
            ingress:
              enabled: true
              hosts:
                - grafana.example.com
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    ```

2. Apply the Grafana Application:

    ```sh
    kubectl apply -f argocd/grafana-application.yaml
    ```

3. ArgoCD will now deploy Grafana using the specified Helm chart. You can monitor the deployment progress in the ArgoCD UI.

4. The service to access grafana:
    Update `/etc/hosts` file by add this line

    ```sh

    ...

    127.0.0.1       grafana.example.com
    ```

    Now you can access the grafana at `http://grafana.example.com:8888`.

5. Login username as `admin` and retrieve the initial password:

    ```sh
    kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```

Now you have Grafana deployed and managed by ArgoCD using a Helm chart. Any updates to the Helm chart will be automatically applied by ArgoCD, ensuring your Grafana installation stays up-to-date.
