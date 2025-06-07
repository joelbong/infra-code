# Gitops on a kubernetes talos cluster in a proxmox environment

## Create nodes in Proxmox -> later with terraform

1. Create a VM and convert to a template.
2. Create 3 control plane nodes from this template
3. Create 3 worker nodes

## Bootstrap talos K8s cluster

1. Install the [talosctl client](https://www.talos.dev/v1.9/talos-guides/install/talosctl/)
2. Generate the secrets (gitignored)
   `talosctl gen secrets -o management-cluster/secrets.yaml --force`
3. Generate template configs (gitignored)
   `talosctl gen config --with-secrets management-cluster/secrets.yaml management-cluster https://${NODE_VIP_DNS_OR_IP}:6443 -o management-cluster/templates/ --force`
4. Create secrets.patch in inline manifest
   - Cilium certs
     - Create helm output for cilium
       `helm template cilium cilium/cilium --version 1.17.4 -n kube-system -f management-cluster/cilium/values.yaml > management-cluster/cilium/helm-output.yaml`
     - remove the part of the cilium certs in the helm-output
     - expose this in public github repo
     - put the generated certs cilium-ca and hubble-server-certs in the management-cluster/patches/secrets.patch
   - [Create secret for proxmox ccm and csi](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/install.md#install-the-plugin-by-using-talos-machine-config)
5. For each control plane node create a talos configuration file (gitignored)
   `talosctl machineconfig patch management-cluster/templates/controlplane.yaml --patch @management-cluster/patches/general.patch --patch @management-cluster/patches/secrets.patch --patch @management-cluster/patches/system-addons.patch --patch @management-cluster/patches/controlplane.patch --patch @management-cluster/patches/${CONTROL_PLANE_NODE}.patch --output management-cluster/nodes-config/${CONTROL_PLANE_NODE}.yaml`
6. For each worker node create a talos configuration file (gitignored)
   `talosctl machineconfig patch management-cluster/templates/worker.yaml --patch @management-cluster/patches/general.patch --patch @management-cluster/patches/system-addons.patch --patch @management-cluster/patches/${WORKER_NODE}.patch --output management-cluster/nodes-config/${WORKER_NODE}.yaml`
7. Apply the config for control plane nodes. IP address will be found on the console
   `talosctl apply-config --insecure --nodes ${NODE_IP} --file management-cluster/nodes-config/${NODE}.yaml`
8. Configure talos endpoints
   `talosctl --talosconfig=./management-cluster/templates/talosconfig config endpoint ${CONTROL_PLANE_NODE_1_IP} ${CONTROL_PLANE_NODE_2_IP} ${CONTROL_PLANE_NODE_3_IP}`
   `talosctl config merge ./management-cluster/templates/talosconfig`
9. Bootstrap cluster
   `talosctl bootstrap --nodes {CONTROL_PLANE_NODE_1_IP}`
10. Get your kubeconfig
    `talosctl config endpoint ${CONTROL_PLANE_NODE_1_IP}`
    `talosctl config node ${CONTROL_PLANE_NODE_1_IP}`
    `talosctl config use-context management-cluster`
    `talosctl kubeconfig`
11. Apply config for worker nodes
    `talosctl apply-config --insecure --nodes ${NODE_IP} --file management-cluster/nodes-config/${NODE}.yaml`

## Prepare gitops with Argocd

1. Cert manager
   `helm repo add cert-manager https://charts.jetstack.io`
   `helm upgrade -i cert-manager cert-manager/cert-manager --version 1.17.2 -n cert-manager --create-namespace -f clusters/in-cluster/cert-manager/helm/cert-manager/values.yaml`
2. External secrets
   1. setup Bitwarden account with secret management
      1. Create a project
      2. Create machine account
      3. Create api token for machine account
   2. Create secret for the bitwarden api token (gitignored)
      `kubectl create ns external-secrets`
      `kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-access-token-secret.yaml`
   3. Create bitwarden certificates
      `kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-server-ca.yaml`
      `kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/bitwarden-server-tls.yaml`
   4. Install chart
      `helm repo add external-secrets-operator https://charts.external-secrets.io/`
      `helm upgrade -i external-secrets external-secrets-operator/external-secrets --version 0.17.0 -n external-secrets --create-namespace -f clusters/in-cluster/external-secrets/helm/external-secrets/values.yaml`
   5. Create clusterstore for bitwarden
      `kubectl apply -n external-secrets -f clusters/in-cluster/external-secrets/resources/clusterstore-bitwarden.yaml`
3. Traefik
   1. Create an API token for your domain in Cloudflare
   2. Put this token in bitwarden as 'cloudflare-api-token'
   3. Install chart
      `helm repo add traefik https://traefik.github.io/charts`
      `helm upgrade -i traefik traefik/traefik --version 36.0.0 -n traefik --create-namespace -f clusters/in-cluster/traefik/helm/traefik/values.yaml`
   4. Apply external-secret for cloudflare-api-token
      `kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-api-token-external-secret.yaml`
   5. Apply issuer with DNS challenge for your domain
      `kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-bongima-dnszone-issuer.yaml`
   6. Apply certificate for all subdomains
      `kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/cloudflare-bongima-subdomains-certificate.yaml`
   7. Apply tls store
      `kubectl apply -n traefik -f clusters/in-cluster/traefik/resources/tls-store.yaml`
4. External-DNS
   1. Create a pihole secret in bitwarden named 'pihole-admin'
   2. Create external-secret for this pihole-admin
      `kubectl create ns external-dns`
      `kubectl apply -n external-dns -f clusters/in-cluster/external-dns/resources/pihole-external-secret.yaml`
   3. Install chart
      `helm repo add external-dns https://kubernetes-sigs.github.io/external-dns`
      `helm upgrade -i external-dns external-dns/external-dns --version 1.16.1 -n external-dns --create-namespace -f clusters/in-cluster/external-dns/helm/external-dns/values.yaml`
5. ArgoCD
   1. Install chart
      `helm repo add argo https://argoproj.github.io/argo-helm`
      `helm upgrade -i argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 8.0.15 -n argocd --create-namespace -f argocd/values.yaml`
   2. Create ingressroute for ArgoCD
      `kubectl apply -n argocd -f clusters/in-cluster/argocd/resources/argocd-ingressroute.yaml`
   3. Create external secret for github repo
      `kubectl apply -n argocd -f clusters/in-cluster/argocd/resources/repository-credential-external-secret.yaml`
   4. Create root argocd application for applicationsets
      `kubectl apply -n argocd -f argocd/root-application.yaml`

## Updates and Maintenances

- Remove dead pods
  `kubectl delete pod --field-selector=status.phase==Succeeded -A`
  `kubectl delete pod --field-selector=status.phase==Failed -A`
- Argocd will auto update the helm charts in clusters
- update Argocd with helm process and update version in argocd/config.yaml
  `helm upgrade -i argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version ${VERSION} -n argocd --create-namespace -f argocd/values.yaml`
- Upgrade talosctl
  `sudo rm /usr/local/bin/talosctl`
  `curl -sL https://talos.dev/install | sh`
- upgrade talos image version
  - get latest image version from [talos image factory](https://factory.talos.dev/)
  - download ISO into proxmox and replace disk in talos VM template
- Upgrade talos image version node per node. Start with controlplane nodes. Continue to next node when previous node in ready state
  `talosctl upgrade --nodes ${NODE_IP/DNS} --image ${TALOS_IMAGE}`
- Update talos machine config
  - update Proxmox csi, Proxmox ccm and Metallb versions in management-cluster/patches/system-addons.patch
  - put new Talos image version in management-cluster/patches/general.patch.
  - Repeat step 3 till step 7 in [talos bootstrap](#bootstrap-talos-k8s-cluster)
- [Upgrade kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- Upgrade kubernetes version. For one controlplane nodes
  `talosctl --nodes ${CONTROL_PLANE_IP/NODE} upgrade-k8s --to ${VERSION}`
- upgrade Proxmox node
