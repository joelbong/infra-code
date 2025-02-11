# Gitops on a kubernetes talos cluster in a proxmox environment

## Create nodes in Proxmox
1. Creat a VM and convert to a template.
2. Create 3 controle plane nodes from this template 
3. Create 3 worker nodes
4. (Optional) Configure for each VM a static IP address for all the control and worker nodes. This was configured in my home router Opnsense with DHCP static mapping (thus based on the mac address of the VM)

## Bootstrap talos K8s cluster
1. Install the [talosctl client](https://www.talos.dev/v1.9/talos-guides/install/talosctl/)
2. Generate the secrets
```talosctl gen secrets -o management-cluster/secrets.yaml```
3. Generate template configs
```talosctl gen config --with-secrets management-cluster/secrets.yaml management-cluster https://NODE_IP:6443 -o management-cluster/templates/```
4. For each control plane node create a talos configuration file
```talosctl machineconfig patch management-cluster/templates/controlplane.yaml --patch @management-cluster/patches/general.patch --patch @management-cluster/patches/${CONTROL_PLANE_NODE}.patch --output management-cluster/nodes-config/${CONTROL_PLANE_NODE}.yaml```
5. For each worker node create a talos configuration file
```talosctl machineconfig patch management-cluster/templates/controlplane.yaml --patch @management-cluster/patches/general.patch --patch @management-cluster/patches/${WORKER_NODE}.patch --output management-cluster/nodes-config/${WORKER_NODE}.yaml```
6. For each node apply the config 
```talosctl apply-config --insecure --nodes ${NODE_IP} --file management-cluster/nodes-config/${NODE}.yaml```
7. Configure talos endpoints
```talosctl --talosconfig=management-cluster/templates/talosconfig config endpoint ${CONTROL_PLANE_NODE_1_IP} ${CONTROL_PLANE_NODE_2_IP} ${CONTROL_PLANE_NODE_3_IP}```
8. Bootstrap cluster
```cp management-cluster/templates/talosconfig ~/.talos/config```
```talosctl bootstrap --nodes {CONTROL_PLANE_NODE_1_IP}```
9. Get your kubeconfig
```talosctl kubeconfig --nodes {CONTROL_PLANE_NODE_1_IP}```

## Install addons on cluster
1. Pre req:
  1. Install helm
2. Cilium
  1. Create priviliged namespace
  ```kubectl create ns cilium-system```
  ```kubectl label ns cilium-system pod-security.kubernetes.io/enforce=privileged```
  2. Install chart
  ```helm repo add cilium https://helm.cilium.io/```
  ```helm upgrade -i cilium cilium/cilium --version 1.17.0 -n cilium-system --create-namespace -f clusters/in-cluster/cilium-system/helm/cilium/values.yaml```
3. Proxmox cloud controller manager, needed to untainted the nodes
   1. Pre-req
    1. (Create a proxmox token)[https://github.com/sergelogvinov/proxmox-cloud-controller-manager/blob/main/docs/install.md#create-a-proxmox-token] for proxmox-cloud-controller-manager
    2. Put this token in bitwarden with secret name 'proxmox-ccm-access-token'. Make sure our machine account has access to it
  2. Create priviliged namespace
  ```kubectl create ns proxmox-ccm-system```
  ```kubectl label ns proxmox-ccm-system pod-security.kubernetes.io/enforce=privileged```
  3. Create secret proxmox-ccm-access-token
  4. Install chart
  ```helm upgrade -i proxmox-ccm oci://ghcr.io/sergelogvinov/charts/proxmox-cloud-controller-manager --version 0.2.11 -n proxmox-ccm-system --create-namespace -f clusters/in-cluster/proxmox-ccm-system/helm/proxmox-ccm/values.yaml```
4. Cert manager
  1. Install chart
  ```helm repo add cert-manager https://charts.jetstack.io```
  ```helm upgrade -i cert-manager cert-manager/cert-manager --version 1.17.0 -n cert-manager --create-namespace -f clusters/in-cluster/cert-manager/helm/cert-manager/values.yaml```
5. Sealed secrets
  1. Install chart
  ```helm repo add bitnami-labs https://bitnami-labs.github.io/sealed-secrets```
  ```helm upgrade -i sealed-secrets bitnami-labs/sealed-secrets --version 2.17.1 -n sealed-secrets --create-namespace -f clusters/in-cluster/sealed-secrets/helm/sealed-secrets/values.yaml```
6. External secrets
  1. Pre-req
    1. Install [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#linux) locally
    2. Bitwarden account with secret management
      1. Create a project
      2. Create machine account
      3. Create api token for machine account
  2. Create a sealed secret for the bitwarden api token
  ```kubeseal --controller-namespace=sealed-secrets --namespace external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-access-token-secret.yaml -w clusters/in-cluster/external-secrets/resources/bitwarden-access-token-sealed-secret.yaml```
  ```kubectl create ns external-secrets```
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-access-token-sealed-secret.yaml```
  3. Create bitwarden certificates
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-server-ca.yaml```
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-server-tls.yaml```
  4. Install chart
  ```helm repo add external-secrets-operator https://charts.external-secrets.io/```
  ```helm upgrade -i external-secrets external-secrets-operator/external-secrets --version 0.14.0 -n external-secrets --create-namespace -f clusters/in-cluster/external-secrets/helm/external-secrets/values.yaml```
  5. Create clusterstore for bitwarden
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/clusterstore-bitwarden.yaml```
7. Proxmox cloud controller manager to create secret proxmox-ccm-access-token with external secret
  1. Create external secret
  ```kubectl apply -n proxmox-ccm-system -f clusters/in-cluster/proxmox-ccm-system/resources/proxmox-ccm-external-secret.yaml```
8. Install proxmox-csi
  1. Pre-req:
    1. (Create a proxmox token)[https://github.com/sergelogvinov/proxmox-cloud-controller-manager/blob/main/docs/install.md#create-a-proxmox-token] for proxmox-csi
    2. Put this token in bitwarden with secret name 'proxmox-csi-access-token'. Make sure our machine account has access to it
    3. Storage configured on Proxmox to be used for proxmox csi
  2. Create priviliged namespace
  ```kubectl create ns proxmox-csi-system```
  ```kubectl label ns proxmox-csi-system pod-security.kubernetes.io/enforce=privileged```
  3. Apply external-secret
  ```kubectl apply -n proxmox-csi-system -f clusters/in-cluster/proxmox-csi-system/resources/proxmox-csi-external-secret.yaml```
  2. Install chart
  ```helm upgrade -i proxmox-csi oci://ghcr.io/sergelogvinov/charts/proxmox-csi-plugin --version 0.3.4 -n proxmox-csi-system --create-namespace -f clusters/in-cluster/proxmox-csi-system/helm/proxmox-csi/values.yaml```
  3. Install default storage
  ```kubectl apply -f clusters/in-cluster/proxmox-csi-system/resources/proxmox-csi-storageclass.yaml```
9. Metallb
  1. Create priviliged namespace
  ```kubectl create ns metallb-system```
  ```kubectl label ns metallb-system pod-security.kubernetes.io/enforce=privileged```
  2. Install chart
  ```helm repo add metallb https://metallb.github.io/metallb```
  ```helm upgrade -i metallb metallb/metallb --version 0.14.9 -n metallb-system --create-namespace```
  3. Create l2-advertisement
  ```kubectl apply -n metallb-system -f clusters/in-cluster/metallb-system/resources/l2-advertisement.yaml```
  4. Create ip pool
  ```kubectl apply -n metallb-system -f clusters/in-cluster/metallb-system/resources/ip-address-pool.yaml```
10. Traefik
  1. Pre-req
    1. Cloudflare account and a domain
      1. Create an API token for your domain in Cloudflare
      2. Put this token in bitwarden as 'cloudflare-api-token'
  2. Install chart
  ```helm repo add traefik https://traefik.github.io/charts```
  ```helm upgrade -i traefik traefik/traefik --version 34.2.0 -n traefik --create-namespace -f clusters/in-cluster/traefik/helm/traefik/values.yaml```
  3. Apply external-secret for cloudflare-api-token
  ```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-api-token-external-secret.yaml```
  4. Apply issuer with DNS challenge for your domain
  ```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-bongima-dnszone-issuer.yaml```
  5. Apply certificate for all subdomains
```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-bongima-subdomains-certificate.yaml```
  6. Apply tls store
  ```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/tls-store.yaml```
11. External-DNS
  1. Pre req
    1. Create a pihole secret in bitwarden named 'pihole-admin'
  2. Create external-secret for this pihole-admin
  ```kubectl create ns external-dns```
  ```kubectl apply -n external-dns -f clusters/in-cluster/external-dns/resources/pihole-external-secret.yaml```
  3. Install chart
  ```helm repo add external-dns https://kubernetes-sigs.github.io/external-dns```
  ```helm upgrade -i external-dns external-dns/external-dns --version 1.15.1 -n external-dns --create-namespace -f clusters/in-cluster/external-dns/helm/external-dns/values.yaml```
12. ArgoCD
  1. Install chart
  ```helm repo add argo https://argoproj.github.io/argo-helm```
  ```helm upgrade -i argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 7.8.2 -n argocd --create-namespace -f clusters/in-cluster/argocd/helm/argocd/values.yaml```
  2. Create ingressroute for ArgoCD
  ```kubectl apply -n argocd -f clusters/in-cluster/argocd/resources/argocd-ingressroute.yaml```


## Gitops with Argocd
1. Create external secret for github repo
```kubectl apply -n argocd -f clusters/in-cluster/argocd/resources/repository-credential-external-secret.yaml```
2. Create root argocd application for applicationsets
```kubectl apply -n argocd -f clusters/in-cluster/argocd/root-application.yaml```
