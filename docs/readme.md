# Gitops on a kubernetes talos cluster in a proxmox environment

## Create nodes in Proxmox -> later with terraform
1. Creat a VM and convert to a template.
2. Create 3 controle plane nodes from this template 
3. Create 3 worker nodes
4. (Optional) Configure for each VM a static IP address for all the control and worker nodes. This was configured in my home router Opnsense with DHCP static mapping (thus based on the mac address of the VM)

## Bootstrap talos K8s cluster
1. Install the [talosctl client](https://www.talos.dev/v1.9/talos-guides/install/talosctl/)
2. Generate the secrets (gitignored)
```talosctl gen secrets -o management-cluster/secrets.yaml```
3. Generate template configs (gitignored)
```talosctl gen config --with-secrets management-cluster/secrets.yaml management-cluster https://NODE_IP:6443 -o management-cluster/templates/```
4. Create [cloud-controller.patch](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/install.md#install-the-plugin-by-using-talos-machine-config) (gitignored)
5. For each control plane node create a talos configuration file (gitignored)
```talosctl machineconfig patch management-cluster/templates/controlplane.yaml --patch @management-cluster/patches/general.patch --patch @management-cluster/patches/proxmox-secrets.patch --patch @management-cluster/patches/system-addons.patch --patch @management-cluster/patches/${CONTROL_PLANE_NODE}.patch --output management-cluster/nodes-config/${CONTROL_PLANE_NODE}.yaml```
6. For each worker node create a talos configuration file (gitignored)
```talosctl machineconfig patch management-cluster/templates/worker.yaml --patch @management-cluster/patches/general.patch --patch @management-cluster/patches/system-addons.patch --patch @management-cluster/patches/${WORKER_NODE}.patch --output management-cluster/nodes-config/${WORKER_NODE}.yaml```
7. Apply the config for controll plane nodes 
```talosctl apply-config --insecure --nodes ${NODE_IP/DNS} --file management-cluster/nodes-config/${NODE}.yaml```
8. Configure talos endpoints
```talosctl --talosconfig=${HOME}/.talos/config config endpoint ${CONTROL_PLANE_NODE_1_IP} ${CONTROL_PLANE_NODE_2_IP} ${CONTROL_PLANE_NODE_3_IP}```
9. Bootstrap cluster
```talosctl bootstrap --nodes {CONTROL_PLANE_NODE_1_IP}```
10. Get your kubeconfig
```talosctl kubeconfig --talosconfig=${HOME}/.talos/config```
11. Apply config for worker nodes
```talosctl apply-config --insecure --nodes ${NODE_IP/DNS} --file management-cluster/nodes-config/${NODE}.yaml```

## Prepare gitops with Argocd 
1. Cert manager
```helm repo add cert-manager https://charts.jetstack.io```
```helm upgrade -i cert-manager cert-manager/cert-manager --version 1.17.1 -n cert-manager --create-namespace -f clusters/in-cluster/cert-manager/helm/cert-manager/values.yaml```
2. External secrets
  1. setup Bitwarden account with secret management
      1. Create a project
      2. Create machine account
      3. Create api token for machine account
  2. Create secret for the bitwarden api token (gitignored)
  ```kubectl create ns external-secrets```
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-access-token-secret.yaml```
  3. Create bitwarden certificates
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-server-ca.yaml```
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-server-tls.yaml```
  4. Install chart
  ```helm repo add external-secrets-operator https://charts.external-secrets.io/```
  ```helm upgrade -i external-secrets external-secrets-operator/external-secrets --version 0.14.2 -n external-secrets --create-namespace -f clusters/in-cluster/external-secrets/helm/external-secrets/values.yaml```
  5. Create clusterstore for bitwarden
  ```kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/clusterstore-bitwarden.yaml```
3. Traefik
  1. Create an API token for your domain in Cloudflare
  2. Put this token in bitwarden as 'cloudflare-api-token'
  3. Install chart
  ```helm repo add traefik https://traefik.github.io/charts```
  ```helm upgrade -i traefik traefik/traefik --version 34.3.0 -n traefik --create-namespace -f clusters/in-cluster/traefik/helm/traefik/values.yaml```
  4. Apply external-secret for cloudflare-api-token
  ```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-api-token-external-secret.yaml```
  5. Apply issuer with DNS challenge for your domain
  ```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-bongima-dnszone-issuer.yaml```
  6. Apply certificate for all subdomains
```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-bongima-subdomains-certificate.yaml```
  7. Apply tls store
  ```kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/tls-store.yaml```
4. External-DNS
  1. Create a pihole secret in bitwarden named 'pihole-admin'
  2. Create external-secret for this pihole-admin
  ```kubectl create ns external-dns```
  ```kubectl apply -n external-dns -f clusters/in-cluster/external-dns/resources/pihole-external-secret.yaml```
  3. Install chart
  ```helm repo add external-dns https://kubernetes-sigs.github.io/external-dns```
  ```helm upgrade -i external-dns external-dns/external-dns --version 1.15.2 -n external-dns --create-namespace -f clusters/in-cluster/external-dns/helm/external-dns/values.yaml```
5. ArgoCD
  1. Install chart
  ```helm repo add argo https://argoproj.github.io/argo-helm```
  ```helm upgrade -i argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 7.8.2 -n argocd --create-namespace -f argocd/values.yaml```
  2. Create ingressroute for ArgoCD
  ```kubectl apply -n argocd -f clusters/in-cluster/argocd/resources/argocd-ingressroute.yaml```
  3. Create external secret for github repo
  ```kubectl apply -n argocd -f clusters/in-cluster/argocd/resources/repository-credential-external-secret.yaml```
  4. Create root argocd application for applicationsets
  ```kubectl apply -n argocd -f argocd/root-application.yaml```
