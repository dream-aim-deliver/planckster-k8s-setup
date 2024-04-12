## Pre Requisites

- Install FluxCD CLI
- Install Kubectl
- Install sops 
- Have access to the Kubernetes cluster where you wish you setup the planckster components
- Make sure you can run kubectl commands on the cluster

## Steps to setup Planckster

For this example, let's assume you are working with Minikube.

1. Start the Minikube cluster
```bash
minikube start --kubernetes-version="v1.25.2"
```

2. Verify the cluster is running
```bash
kubectl get nodes

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   69s   v1.25.2

```

3. Install FluxCD CLI ( v2.1.0 ), you might have to download this version from the releases page on GitHub
```bash
brew install fluxcd/tap/flux
```

4. Generate the FluxCD manifests
```bash
mkdir -p clusters/minikube
cd clusters/minikube

flux install --version=v2.1.0 --export > flux.yaml
# wget https://github.com/fluxcd/flux2/releases/download/v2.1.0/install.yaml
# mv install.yaml flux.yaml
```

5. Apply the FluxCD manifests
```bash
kubectl apply -f flux.yaml
```

6. Verify the FluxCD components are running
```bash
watch kubectl get pods -n flux-system
```

7. Generate a private Age Key or skip this step if you already have one

```bash
age-keygen -o age.agekey
```

** NOTE ** Make sure to keep the age.agekey file safe and secure. Save the public key to `sops.pub`.
Now is also a good time to copy the public key to the kubesat-planckster repository and encrypt the secrets.


8. Create a secret for the Age Key
```bash
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```


9. Generate the Kustomize manifest and Source manifest for Kubesat-Planckster
```bash
flux create source git kubesat-planckster \
  --url=https://github.com/dream-aim-deliver/kubesat-planckster \
  --branch=main \
  --export > kubesat-planckster.yaml

flux create kustomization infrastructure \
  --source=GitRepository/kubesat-planckster \
  --path="./infrastructure" \
  --prune=true \
  --decryption-provider=sops \
  --decryption-secret=sops-age \
  --interval=1m \
  --export >> kubesat-planckster.yaml
```

Set the RELEASE_PATH variable to `./releases/staging` or `./releases/production`. It indicates where the manifests are located in the kubesat-planckster repository.

```bash
export RELEASE_PATH="./releases/staging"
```

Double check the path to the release manifests in the kubesat-planckster repository.
```bash
echo $RELEASE_PATH
```
Then create the kustomization for the release manifests

```bash
flux create kustomization release \
  --source=GitRepository/kubesat-planckster \
  --path="$RELEASE_PATH" \
  --prune=true \
  --interval=1m \
  --depends-on=infrastructure \
  --decryption-provider=sops \
  --decryption-secret=sops-age \
  --export >> kubesat-planckster.yaml
```

10. Create the namespace for the application
```bash
kubectl create namespace sda
```

11. Apply the manifests
```bash
kubectl apply -f kubesat-planckster.yaml
```

12. Verify the components are running
```bash
kubectl get pods -n sda
```


