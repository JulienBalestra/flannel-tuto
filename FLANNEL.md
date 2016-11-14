# Image

The concept of the image is the same with DRY in programming.

Install everything you could used at least two times.


    sudo -i
    apt-get update -q
    apt-get upgrade -y
    apt-get install -y curl tar
    
### Docker
    
    curl -L https://get.docker.com | sh
    sudo docker pull quay.io/coreos/etcd
    sudo docker pull quay.io/coreos/flannel:0.5.5
    
 
### Rkt
    
    # https://github.com/coreos/rkt/releases
    mkdir -v /tmp/rkt_install || rm -Rf /tmp/rkt_install/*
    mkdir -pv /usr/lib/rkt/stage1-images    
    
    curl -o /tmp/rkt_install/rkt.tar.gz \
        -L https://github.com/coreos/rkt/releases/download/v1.19.0/rkt-v1.19.0.tar.gz
    cd -P /tmp/rkt_install
    tar -xzvf /tmp/rkt_install/rkt.tar.gz --strip-components=1
    mv -v /tmp/rkt_install/rkt /usr/bin/rkt
    mv -v /tmp/rkt_install/*.aci /usr/lib/rkt/stage1-images/
    mv -v /tmp/rkt_install/bash_completion/rkt.bash /usr/share/bash-completion/completions/rkt
    rkt version
    groupadd rkt
    
    # AS ROOT
    cat << EOF > /etc/rkt/net.d/10-flannel.conf
    {
      "name": "flannel",
      "type": "flannel",
      "delegate": {
        "isDefaultGateway": true,
        "ipMasq": true
      }
    }
    EOF
    rkt fetch --insecure-options=all quay.io/coreos/alpine-sh
    
    
## Passwd

    sudo passwd ${myuser}
    
## Clone the VirtualMachine to get an Image


## Etcd 

Etcd is a Key Value store who can distribute the data accross several nodes calls Members:

* 1
* 3
* 5
* 9

And have Etcd Proxy / Proxies with the same Client REST API as the members.

To stay simple we are here just making a single cluster node and one or many proxies.


## Virtual Machine One

   
    # Fill the PUBLIC_IP= by the external reachable IPv4 -> ip a
    
    PUBLIC_IP=I.P.I.P    
    docker run --rm --net=host -it \
        -e ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379 \
        -e ETCD_ADVERTISE_CLIENT_URLS=http://${PUBLIC_IP}:2379 \
        quay.io/coreos/etcd
        
    # Be sure by running this:
     
    curl -fsL http://127.0.0.1:2379/health
    # should return 0 and stdout: {"health": "true"}
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl cluster-health
    # The same but with the etcdctl
    
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl \
        set /coreos.com/network/config \
        '{"Network":"10.1.0.0/16", "Backend": {"Type": "vxlan"}}'
            
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl ls -r
    # /coreos.com
    # /coreos.com/network
    # /coreos.com/network/config
          
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl \
            get /coreos.com/network/config
    # {"Network":"10.1.0.0/16", "Backend": {"Type": "vxlan"}}

    docker run --privileged=true --rm --net=host \
        -v /dev/net/tun:/dev/net/tun:ro \
        -v /run/flannel:/run/flannel \
        quay.io/coreos/flannel:0.5.5 /opt/bin/flanneld --ip-masq=true
        
    rkt run --interactive --net=flannel quay.io/coreos/alpine-sh --exec /bin/sh
    ip a
    # should be an IP address in prefix 10.1.0.0/16
    # CTRL + D
    
    # On the host
    ip a
    # With flannel.1 and cni0
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl ls -r
    # Should be /coreos.com/network/subnets/10.1.XX.0/16 == flannel.1
    # Should be /coreos.com/network/subnets/10.1.XX.1/24 == cni0
    
    # Again re run alpine-sh
    rkt run --interactive --net=flannel quay.io/coreos/alpine-sh --exec /bin/sh
    ip a
    # note the Flannel IP address and stay in the container
    
## Virtual Machine Two

* Needed
    * PUBLIC_IP of the remote Virtual Machine One: For the Etcd access
    
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl \
        -C http://${PUBLIC_IP}:2379 cluster-health

    # member 8e9e05c52164694d is healthy: got healthy result from http://${PUBLIC_IP}:2379
    # cluster is healthy
    
    docker run --privileged=true --rm --net=host \
        -v /dev/net/tun:/dev/net/tun:ro \
        -v /run/flannel:/run/flannel \
        quay.io/coreos/flannel:0.5.5 \
        /opt/bin/flanneld --ip-masq=true \
        -etcd-endpoints=http://192.168.132.135:2379
    
    ip a
    # See the flannel.1
    docker run --rm --net=host -it quay.io/coreos/etcd etcdctl \
            -C http://${PUBLIC_IP}:2379 ls -r
    # /coreos.com
    # /coreos.com/network
    # /coreos.com/network/config
    # /coreos.com/network/subnets
    # /coreos.com/network/subnets/10.1.18.0-24
    # /coreos.com/network/subnets/10.1.26.0-24
    # Now there are two entries in the subnet directory
    
    docker run --net=host quay.io/coreos/etcd etcdctl \
        -C http://${PUBLIC_IP}:2379 \
        get /coreos.com/network/subnets/10.1.18.0-24 | \
        jq -r .PublicIP | xargs ping -c 1

    rkt run --interactive --net=flannel quay.io/coreos/alpine-sh --exec /bin/sh
    ip a
    # should be an IP address in prefix 10.1.0.0/16  
    ping ${THE_OTHER_REMOTE_ALPINE_SH}
    
    

