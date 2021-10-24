# homeserver-k8s
Basic repo to remind me how to stand up k8s on my single home server for testing.

## Enable net.bridge.bridge-nf-call-iptables
    cat > /etc/sysctl.d/20-bridge-nf.conf <<EOF
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl --system
    
Now we gotta match docker and kubelet cgroups, so we create the following dir & file and paste the content prior to install

    mkdir -p /etc/docker
    vim /etc/docker/daemon.json
    
Now paste the following in the daemon.json file:

    {
      "exec-opts": ["native.cgroupdriver=systemd"]
    }
    
## Install Docker with recommended settings
See here for details: https://docs.docker.com/engine/install/ubuntu/

     sudo apt-get update && sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release
     curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
     sudo apt-get update
     sudo apt-get install docker-ce docker-ce-cli containerd.io
     
## Install kubernetes required components:
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    
# Now we install k8s with kubeadm
Please keep in mind I use 192.168.0.0 here as my home network does not conflict you may decide to use something else here but be sure to update calico also or your CNI of choice

    sudo kubeadm init --pod-network-cidr=192.168.0.0/16

## Now we set up kubectl config files so we can talk to k8s directly

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Install cillium (works well with metallb)

    These configurations are both broken and cause my server to stop responding (Inet goes down...)
    presumably because the pod service is overlapping with my actual network... need to fix this later...

    
    helm repo add cilium https://helm.cilium.io/
    helm repo update

    helm install cilium cilium/cilium --version 1.10.5 \
    --namespace kube-system \
    --set kubeProxyReplacement=strict \
    --set k8sServiceHost=10.0.0.254 \
    --set k8sServicePort=6443 \
    --set hubble.listenAddress=":4244" \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"

helm install cilium cilium/cilium --version 1.10.5 \
    --namespace kube-system \
    --set config.ipam=kubernetes \
    --set native-routing-cidr=10.0.0.0/24 \
    --set global.ipMasqAgent.enabled=true \
    --set global.kubeProxyReplacement=strict \
    --set global.k8sServiceHost=10.0.0.254 \
    --set global.k8sServicePort=6443

watch kubectl get pods -n kube-system

# Now we wait till calico is done then taint the masters so we can run pods on them

    kubectl taint nodes --all node-role.kubernetes.io/master-

# Now we install metal LB for load balancer services

Create the following config map and please make sure to update the addresses that are available in your dhcp server.  These should not conflict with things in your actual network ever.

    apiVersion: v1
    kind: ConfigMap
    metadata:
    namespace: metallb-system
    name: config
    data:
    config: |
        address-pools:
        - name: default
          protocol: layer2
          addresses:
          - 10.0.0.201-10.0.0.250

Now you will have load balancing available over those IP's
    
    
## Now we install helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

First be sure to go here: https://helm.sh/docs/intro/install/ (Installing Helm)

Then we install nginx via helm and set the node ports for the local server:

    helm repo update

    helm install ingress-nginx ingress-nginx/ingress-nginx

    kubectl wait --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
    
## External settings

After nginx ingress is completed lets retrieve the service ip:

    $ kubectl get svc nginx
    
    NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    nginx   LoadBalancer   10.107.55.200   10.0.0.201    80:32080/TCP,443:32443/TCP   1m 

Now the final thing to do is create a static DNS entry for your new nginx ingress load balancer and point all your applicable ingresses to it.
    
