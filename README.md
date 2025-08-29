# Server setup

This is my current kubernetes based setup used for my infrastructure.

## ToDo

find solutions for

- provider for PVs and PVCs (Longhorn?)
- backup
  - used software (velero, kasten k10 or something else?)
  - used storage backend (restic or kopia?)
- alternative to watchtower (update used image if there is a newer version fo the same tag)

- ensure the control plane is only reachable locally
- configure firewall or iptables to prevent outgoing traffic in 10.0.0.0/8 and incoming traffic to the control plane (<server IP>:6443)

## Prepare the server

We start with fresh installation of Debian 12 (since that's what's available from my hoster without additional cost).

Fill in placeholders with the appropriate values. Keep in mind that this is very much opinionated.

### Basics setup

1. refresh available packages: `apt update`
2. install dependencies and useful things: `apt install sudo htop git`
3. install updates: `apt upgrade`
4. add user: `useradd -m <username>`
5. make superuser: `usermod -a -G sudo <username>`
6. allow passwordless sudo: use `visudo` to add the following line: `%sudo ALL=(ALL:ALL) NOPASSWD: ALL`
7. set password for user: `passwd <username>`, enter new password
8.reboot with `sudo reboot now` and log in with your new user
8. copy your public key to the user account: from your host (not the server!) run `ssh-copy-id -i public_key.pub <username>@<server address>`, you might have to also add the `-f` flag
9. edit `/etc/ssh/sshd_config` to contain the following lines:
   - `PermitRootLogin no`
   - `PasswordAuthentication no`
10. generate an ssh key pair for the machine: `ssh-keygen`
11. restart the ssh daemon: `sudo systemctl restart sshd`

### A bit of customisation

1. install zsh and related bits: `sudo apt install zsh powerline fonts-powerline`
2. install antidote (zsh plugin manager): `git clone --depth=1 https://github.com/mattmc3/antidote.git ~/.antidote`
3. edit zsh config: `nano .zshrc` (my base config is in the repo)
4. add file containing the zsh plugins: `nano .zsh_plugins.txt` (mine is in the repo)
5. run zsh for the first time, the default config is fine for now: `zsh`
6. make it the default shell: `sudo usermod -s /usr/bin/zsh $(whoami)`

### Basic hardening of the server

1. install fail2ban: `sudo apt install fail2ban`, it should be active after a reboot
   - check if it is running without issues: `systemctl status fail2ban.service` and `sudo journalctl -u fail2ban`
   - if it crashes since it can't find any logs: try adding `backend = systemd` to `/etc/fail2ban/jail.d/defaults-debian.conf`
2. install unattended-upgrades: `sudo apt install unattended-upgrades apt-listchanges`
3. enable it: `sudo dpkg-reconfigure unattended-upgrades` and select "Yes"

## Setup kubernetes

We will use containerd as container runtime. We installed it with docker earlier. The docker daemon will be available on the system as well to allow us to use the new type of Nextcloud apps (called ExApp) which require a connection to the host docker proxy (via another container).

### Prepare the host

1. install docker as described here: https://docs.docker.com/engine/install/debian/#install-using-the-repository
2. ensure that the kernel modules overlay and br_netfilter are enabled. Check their status with `lsmod | egrep 'br_netfilter|overlay'` and enable them with `sudo modprobe overlay` and `sudo modprobe br_netfilter`
3. set kernel parameters in /etc/sysctl.d/kubernetes.conf (you can check their current state with `sysctl <parameter name>`):
   ```
   # process traffic through bridges in iptables 
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   # enable ip forwarding
   net.ipv4.ip_forward = 1
   ```
4. reload kernel config with `sudo sysctl --system`
5. configure containerd to use systemd as cgroup driver by
   1. setting the containerd config to its default with containerd config default > /etc/containerd/config.toml and
   2. changing the parameter "SystemdCgroup" to true in the section "[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]"
   3. restart container with `sudo systemctl restart containerd`
   4. make sure the systemd service for containerd is enabled (with `systemctl status containerd`)

### Create the cluster

1. add an entry for the cluster in the hosts file (with `sudo nano /etc/hosts`) like this: `127.0.0.1 k8s-endpoint`
2. install the required components as described here: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
   - kubelet, kubeadm and kubectl on the server
   - kubeadm and kubectl on the host
   - !!! I did not pin the version of the kubelet and related tools (i.e. skipped the `sudo apt-mark hold …`) to keep them updated automatically. Be aware that this comes at the cost of availability and stability.
   - in this specific case it boils down to these steps:
     1. `sudo apt-get install -y apt-transport-https ca-certificates curl gpg` to install dependencies
     2. `curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg` to add the public signing key for the repo
     3. `echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list` to add/replace the kubernetes repo
     4. `sudo apt-get update` to refresh the package list
     5. `sudo apt-get install -y kubelet kubeadm kubectl` to install the packages
     6. `sudo systemctl enable --now kubelet` to run the kubelet
3. copy the file "kubeadm-config.yaml" from the repo to the server, insert appropriate values into placeholders
4. init the kubelet with `sudo kubeadm init --config kubeadm-config.yaml`
5. copy the kubectl config as described in the output of the previous step

### Install tools

1. install helm as described here: https://helm.sh/docs/intro/install/
   - in short:
     1. `curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null`
     2. `sudo apt-get install apt-transport-https --yes`
     3. `echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list`
     4. `sudo apt-get update`
     5. `sudo apt-get install helm`
2. install k9s as described in https://k9scli.io/topics/install/
    - in short:
      1. copy the URL to the fitting version from https://github.com/derailed/k9s/releases
      2. download the .deb with `curl -L <URL IN HERE> -o k9s_linux_amd64.deb`
      3. install it with `sudo dpkg -i k9s_linux_amd64.deb`

### Install essentials

1. install Cilium as described here: https://docs.cilium.io/en/stable/installation/k8s-install-helm/ but set the option "operator.replicas" to 1 as shown here:
   1. add the helm repo with `helm repo add cilium https://helm.cilium.io/`
   2. install cilium (replace the version with the current one!) with `helm install cilium cilium/cilium --version 1.17.4 --namespace kube-system --set operator.replicas=1`

## Deploy applications to the cluster

### Initial deployment[s.yaml](../../s.yaml)

1. Deploy Argo CD with `kubectl apply -k k8s-deployments/cluster-management/argo-cd/_overlays/<env name>`
2. Deploy remaining applications with `kubectl apply -k k8s-deployments/_overlays/<env name>`

### Connect to secrets stored in Scaleway

This assumes that you already have an account on Scaleway as well as a project for this, an application to access the project as well as reasonable restrictions on the application (like read-only access to resources; keep in mind that global read-only access is not enough, the application also has to be able to read the content of secrets).

1. create a new API key for your existing application and use it to create a Kubernetes secret as follows:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     namespace: external-secrets
     name: scaleway-credentials
   stringData:
     keyId: '<key ID goes here>'
     secretKey: '<secret key generated by Scaleway goes here>'
   ```
2. wait a bit until the ClusterSecretStore is ready



## TODO:

- prevent access to k8s API from outside the server

## Misc

### Credits

- the following guide was very helpful in solving some of the issues I encountered: https://facsiaginsa.com/kubernetes/install-kubernetes-single-node
