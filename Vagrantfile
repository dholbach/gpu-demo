PATCHED_KUBEADM = "/home/vagrant/kubeadm/kubeadm"

Vagrant.configure("2") do |config|
    config.vm.box = "bento/ubuntu-18.04"
  
    config.vm.provider "virtualbox" do |vb|
      vb.cpus   = "2"
      vb.memory = "2048"
    end
  
    apt_docker = "18.09.7-0ubuntu1~18.04.4"
    apt_k8s    = "1.16.2-00"
    v_k8s      = "1.16.2"

    config.vm.synced_folder \
        "/home/daniel/go/src/github.com/kubernetes/kubernetes/bazel-bin/cmd/kubeadm/linux_amd64_pure_stripped", \
        "/home/vagrant/kubeadm"

    config.vm.provision "shell", privileged: true, inline: <<-SHELL
        if [ ! -d ${PATCHED_KUBEADM} ]; cp ${PATCHED_KUBEADM} /usr/local/bin; fi
            chmod a+x /usr/local/bin/kubeadm
        SHELL
    

    num_controlplane = 1 # at the moment, scripts only support 1
    num_workers      = 1
  
    install = <<~SHELL
      set -ex
      export DEBIAN_FRONTEND=noninteractive
      aptGet='apt-get -q'
      # Make sure curl and apt SSL support is available
      ${aptGet} update && ${aptGet} install -y apt-transport-https curl

      # Add the Kubernetes apt signing key and repository
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
      echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
      # Install kubelet, kubectl and docker
      ${aptGet} update && ${aptGet} install -y \
        docker.io=#{apt_docker} \
        kubelet=#{apt_k8s} \
        kubectl=#{apt_k8s}
      # Disable swap, it must not be used when Kubernetes is running
      swapoff -a
      sed -i /swap/d /etc/fstab
      # Enable the docker systemd service
      systemctl enable docker.service
      # Pre-configure the kubelet to bind to eth1
      #   ref: https://github.com/kubernetes/kubeadm/issues/203#issuecomment-335416377
      eth1_ip=$(ifconfig eth1 | awk '$1 == "inet" {print $2}')
      echo "Environment=\"KUBELET_EXTRA_ARGS=--node-ip=${eth1_ip}\"" >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
      # Route the cluster CIDR via eth1
      #   ref: https://github.com/kubernetes/kubeadm/issues/102#issuecomment-291532883
      cat << EOF > /etc/netplan/90-k8s-services-eth1.yaml
      ---
      network:
        version: 2
        ethernets:
          eth1:
            routes:
            - to: 10.96.0.0/16
              via: 0.0.0.0
      EOF
      netplan apply
    SHELL
  
    kubeadm_init = <<~SHELL
      set -ex
      # Configure kubectl for the kubeadm kubeconfig
      export KUBECONFIG=/etc/kubernetes/admin.conf
      cat << EOF >> /etc/bash.bashrc
        export KUBECONFIG=/etc/kubernetes/admin.conf
        alias k="kubectl"
        alias ks="kubectl -n kube-system"
      EOF
      # Init the cluster
      eth1_ip=$(ifconfig eth1 | awk '$1 == "inet" {print $2}')
      stat $KUBECONFIG || kubeadm init \
        --kubernetes-version v#{v_k8s} \
        --apiserver-advertise-address "${eth1_ip}" \
        --pod-network-cidr 10.96.0.0/16 \
        --token abcdef.0123456789abcdef
      # Install Weave Net as the Pod networking solution
      # Workaround  https://github.com/weaveworks/weave/issues/3700
      ### kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 -w0)"
      kubectl apply -f "https://raw.githubusercontent.com/weaveworks/weave/master/prog/weave-kube/weave-daemonset-k8s-1.9.yaml"
      # Make this control plane node able to run normal workloads
      kubectl taint nodes --all node-role.kubernetes.io/master- || true # fails if already untainted
    SHELL
  
    kubeadm_join_async = <<~SHELL
      set -ex
      # Join the cluster asynchronously
      nohup sh -c "
        until kubeadm join \\
          --token abcdef.0123456789abcdef \\
          --discovery-token-unsafe-skip-ca-verification \\
          192.168.5.#{10 + 1}:6443
        do sleep 2
        done
      " >> /var/log/kubeadm_join_async.log 2>&1 & disown
    SHELL
  
    check = <<~SHELL
      set -ex
      echo "Waiting for all nodes to join..."
      export KUBECONFIG=/etc/kubernetes/admin.conf
      num_expected="#{ [0,num_controlplane].max + [0,num_workers].max }"
      if timeout 300 sh -c "
          until kubectl get nodes -oname | wc -l | grep "^$num_expected$"
          do echo -n .; sleep 2
          done
        "
      then echo "All nodes joined the cluster"
      else echo "Timed out waiting for all nodes to join the cluster" 1>&2 && exit 1
      fi
      echo "Waiting for all nodes to be ready..."
      if timeout 100 sh -c "
          while kubectl get nodes | grep NotReady
          do echo -n .; sleep 2
          done
        "
      then echo "All nodes are now Ready"
      else echo "Timed out waiting for all nodes to become Ready" 1>&2 && exit 2
      fi
      echo "Waiting for all pods to run..."
      if timeout 100 sh -c "
          while kubectl get pods --all-namespaces | grep ContainerCreating
          do echo -n .; sleep 2
          done
        "
      then echo "All pods are now running"
      else echo "Timed out waiting for all pods to run" 1>&2 && exit 2
      fi
    SHELL
  
    config.vm.provision "shell", inline: install
  
    (1..num_workers).each do |i|
      config.vm.define "worker-#{i}" do |worker|
        worker.vm.hostname = "worker-#{i}"
        worker.vm.network "private_network", ip: "192.168.5.#{100 + i}"
        worker.vm.provision "shell", inline: kubeadm_join_async
      end
    end
  
    (1..num_controlplane).each do |i|
      config.vm.define "controlplane-#{i}" do |controlplane|
        controlplane.vm.hostname = "controlplane-#{i}"
        controlplane.vm.network "private_network", ip: "192.168.5.#{10 + i}"
        controlplane.vm.provision "shell", inline: kubeadm_init
        controlplane.vm.provision "shell", inline: check
      end
    end
  
  end

  # expose 8080
  # copy over kube config