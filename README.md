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
    
## I prefer calico as my CNI as thats what I use in prod so install tigera operator
See howto here: https://docs.projectcalico.org/getting-started/kubernetes/quickstart

    kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
    kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml

Now we watch calico pods to make sure they go into `RUNNING` first before proceding

    watch kubectl get pods -n calico-system

# Now we wait till calico is done then taint the masters so we can run pods on them

    kubectl taint nodes --all node-role.kubernetes.io/master-
    
# Installing NGinx ingress controller as bare metal install:

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.2/deploy/static/provider/baremetal/deploy.yaml
    
# Install cert manager

    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
    

# Now we create a dummy http service to test everything against:
Make the `dummy.yml` file which we can kubectl create next

    apiVersion: v1
    kind: Service
    metadata:
      name: echo1
    spec:
      ports:
      - port: 80
        targetPort: 5678
      selector:
        app: echo1
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: echo1
    spec:
      selector:
        matchLabels:
          app: echo1
      replicas: 2
      template:
        metadata:
          labels:
            app: echo1
        spec:
          containers:
          - name: echo1
            image: hashicorp/http-echo
            args:
            - "-text=echo1"
            ports:
            - containerPort: 5678
            
Now save it and exit then:

    kubectl apply -f dummy.yml
    
    
## Now we craete the stub ingress for the echos
Make a file called `echo_ingress.yml`  make sure to update echo1.example.com with your own local domain that you use in your home network.

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-echo
      annotations:
        # use the shared ingress-nginx
        kubernetes.io/ingress.class: "nginx"
    spec:
      rules:
      - host: echo1.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo1
                port:
                  number: 80

