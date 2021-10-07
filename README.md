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
    
    
## Now we install helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

First be sure to go here: https://helm.sh/docs/intro/install/ (Installing Helm)

Then we install nginx via helm and set the node ports for the local server:

    helm repo update

    helm install ingress-nginx ingress-nginx/ingress-nginx --set controller.service.type=NodePort --set controller.service.httpPort.nodePort=32080 --set controller.service.httpsPort.nodePort=32443

    kubectl wait --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
    
## Setup the server to listen on port 80 / 443
Since NodePort is a high selected port range on the box you still have to listen to it on the local box via another means.  I prefer using nginx for this as I use nginx on the box for other things beyond tcp port 80/443 binding and forwarding to k8s.  You can obviously use something like socat or anything else but please don't try to update the nodeports to low values during k8s install by overriding the standard ports.

We will become root for this as we are adding services to systemd that will be running the redirects for us:

    sudo apt install -y nginx

Now that we have a local install of nginx lets update the config file to look like this:

    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    include /etc/nginx/modules-enabled/*.conf;

    events {
            worker_connections 768;
            # multi_accept on;
    }

    stream {
        # ...
        server {
            listen     80;
            proxy_pass 127.0.0.1:32080;
        }
        server {
            listen     443;
            #TCP traffic will be forwarded to the specified server
            proxy_pass 127.0.0.1:32443;
        }
    }
    
    
Notice that you can update port 32080 to your nodeport that you set during the helm install for http port and also update 32443 for your https port, then simply start and enable nginx with:

    sudo systemctl enable nginx
    sudo service nginx start
    
