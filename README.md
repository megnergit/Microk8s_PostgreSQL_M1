# PostgreSQL on microk8s

## Motivation

We aim to conduct experiments running PostgreSQL on a microk8s cluster. Our goals include:

- Evaluating whether it is reasonable to run PostgreSQL on a microk8s cluster.
- Setting up a high-availability configuration with one primary and two secondaries.
- Implementing automatic failover using Stolon.
- Utilizing external persistent storage provided by Longhorn.

## Overview

1. Create a VM
2. Install microk8s on the VM
3. Deploy Longhorn on microk8s
4. Deploy PostgreSQL on microk8s with LoadBalancer Stolon
5. Test failover of PostgreSQL

---

## 3. Deploy Longhorn on microk8s
### Tasks Overview

1. Enable the Helm 3 add-on for microk8s.
2. Deploy Longhorn using Helm.
3. Modify the Longhorn deployment to set the `kubelet` directory.
4. Set up `kubectl port-forward` for the Longhorn dashboard.
5. Access the Longhorn dashboard from the host.

### Details

#### 1. Enable Helm 3 add-on for microk8s

On **VM**:
```sh
microk8s enable helm3
```

Verify that `helm3` is enabled:
```sh
microk8s.helm version
```

For convenience, create an alias:
```sh
alias helm=microk8s.helm
```

#### 2. Deploy Longhorn using Helm

Add the Longhorn Helm repository:
```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Install Longhorn on the microk8s cluster:
```sh
helm install longhorn longhorn/longhorn \
             --namespace longhorn-system \
             --create-namespace
```

If something goes wrong, roll back the installation:
```sh
helm uninstall longhorn -n longhorn-system
```
or delete the entire namespace:
```sh
kubectl delete ns longhorn-system
```

To clean up the Custom Resource Definitions (CRDs):
```sh
kubectl delete crd $(kubectl get crd | grep longhorn | awk '{print $1}')
```

Remove log directories:
```sh
sudo rm -rf /var/lib/longhorn/
```

Then restart the cluster:
```sh
microk8s stop
microk8s start
```

#### 3. Modify the Longhorn deployment for `kubelet` directory

Check if Longhorn is running. This can also be done from the host:
```sh
kubectl config set-context --current --namespace longhorn-system
kubectl get po
```

If `longhorn-driver-deployer` is not working, check the logs:
```sh
kubectl logs longhorn-driver-deployer-XXX
```

The `longhorn-driver-deployer` may fail to locate the `kubelet` root directory. To fix this, update the deployment.

Save the configuration of `longhorn-driver-deployer`:
```sh
kubectl get deployments longhorn-driver-deployer -o yaml > longhorn-driver-deployer.yaml
```

Add the last line below:
```yaml
    spec:
      containers:
      - command:
        - longhorn-manager
        - -d
        - deploy-driver
        - --manager-image
        - longhornio/longhorn-manager:v1.8.0
        - --manager-url
        - http://longhorn-backend:9500/v1
        - --kubelet-root-dir=/var/lib/kubelet
```

Apply the update:
```sh
kubectl replace -f longhorn-driver-deployer.yaml --force
```

Check if `longhorn-driver-deployer` is now running:
```sh
kubectl get po
```

#### 4. Set up `kubectl port-forward` for the Longhorn dashboard

To access the Longhorn dashboard from the host, set up port forwarding.
Execute the following command on either the **host** or **VM**:
```sh
kubectl port-forward -n longhorn-system service/longhorn-frontend 8080:80
```

This forwards port 80 of the `longhorn-frontend` service (inside the `longhorn-system` namespace) to port 8080 on the host.

#### 5. Access the Longhorn dashboard from the host

Open the following URL in a web browser on the **host**:
```sh
http://localhost:8080
```

If everything is set up correctly, the Longhorn dashboard should be accessible.

![Longhorn dashboard](./images/longhorn-1.png)

Great!

---

## 4. Deploy PostgreSQL on microk8s with LoadBalancer Stolon

## 5. Test failover of PostgreSQL

### Tasks Overview

---

## What did not work out / Mistakes

1. **Ubuntu Desktop → Ubuntu Server**
2. **Minimum requirements:**
   - **2 vCPU**
   - **4 GiB RAM**
   - **20 GiB storage**

Make sure you have installed **Ubuntu Server** after setting up the VM:

```sh
lsb_release -a
cat /etc/*release*
```

---

# END

