# maas-juju-k8s

### install maas
```
sudo snap install --channel=latest/stable lxd
sudo snap refresh --channel=latest/stable lxd
sudo snap install jq
sudo snap install maas
sudo snap install maas-test-db
```

### clone repository
```
cd ~
git clone https://github.com/anggakg/maas-juju-k8s.git
```

### get local interface name
```
export INTERFACE=$(ip route | grep default | cut -d ' ' -f 5)
export IP_ADDRESS=$(ip -4 addr show dev $INTERFACE | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p
sudo iptables -t nat -A POSTROUTING -o $INTERFACE -j SNAT --to $IP_ADDRESS
```

### set persistent nat conf
```
echo iptables-persistent iptables-persistent/autosave_v4 boolean true | sudo debconf-set-selections
echo iptables-persistent iptables-persistent/autosave_v6 boolean true | sudo debconf-set-selections
sudo apt-get install iptables-persistent -y
```

### LXD init
```
sudo cat maas-juju-k8s/lxd.conf | lxd init --preseed
```

### verify LXD network config
```
lxc network show lxdbr0
lxd waitready
```

### Initialise MAAS
```
sudo maas init region+rack --database-uri maas-test-db:/// --maas-url http://${IP_ADDRESS}:5240/MAAS
sleep 30
```

### Create MAAS admin and grab API key
```
sudo maas createadmin --username admin --password admin --email admin
export APIKEY=$(sudo maas apikey --username admin)
```

### MAAS login admin
```
maas login admin 'http://localhost:5240/MAAS/' $APIKEY
```

### Configure MAAS networking (set gateways, vlans, DHCP on etc).
```
export SUBNET=10.10.10.0/24
export FABRIC_ID=$(maas admin subnet read "$SUBNET" | jq -r ".vlan.fabric_id")
export VLAN_TAG=$(maas admin subnet read "$SUBNET" | jq -r ".vlan.vid")
export PRIMARY_RACK=$(maas admin rack-controllers read | jq -r ".[] | .system_id")
maas admin subnet update $SUBNET gateway_ip=10.10.10.1
maas admin ipranges create type=dynamic start_ip=10.10.10.200 end_ip=10.10.10.254
maas admin vlan update $FABRIC_ID $VLAN_TAG dhcp_on=True primary_rack=$PRIMARY_RACK
maas admin maas set-config name=upstream_dns value=8.8.8.8
```

### Add LXD as a VM host for MAAS and capture the VM_HOST_ID
```
export VM_HOST_ID=$(maas admin vm-hosts create  password=password  type=lxd power_address=https://${IP_ADDRESS}:8443 \
 project=maas | jq '.id')
```

### allow high CPU oversubscription so all VMs can use all cores
```
maas admin vm-host update $VM_HOST_ID cpu_over_commit_ratio=8
```

### create tags for MAAS
```
maas admin tags create name=juju-controller comment='This tag should to machines that will be used as juju controllers'
maas admin tags create name=metal comment='This tag should to machines that will be used as bare metal'
```

### add a VM for the juju controller
```
maas admin vm-host compose $VM_HOST_ID cores=8 memory=4096 architecture="amd64/generic" \
storage="main:25(pool1)" hostname="juju-controller"
# add tag
export JUJU_SYSID=$(maas admin machines read | jq  '.[] 
| select(."hostname"=="juju-controller") 
| .["system_id"]' | tr -d '"')
maas admin tag update-nodes "juju-controller" add=$JUJU_SYSID
```

### juju setup
```
cd ~
sudo snap install juju --classic
sed -i "s/IP_ADDRESS/$IP_ADDRESS/" maas-juju-k8s/maas-cloud.yaml
juju add-cloud --local maas-cloud maas-juju-k8s/maas-cloud.yaml
juju add-credential maas-cloud
juju clouds --local
juju credentials
# Bootstrap the maas-cloud
juju bootstrap maas-cloud --bootstrap-constraints "tags=juju-controller mem=4G"
# open dashboard
juju dashboard
```

### download charmed-kubernetes (optional)
```
juju download charmed-kubernetes --channel latest/stable --filepath charmed-kubernetes-latest-stable.bundle
unzip -q charmed-kubernetes-latest-stable.bundle -d charmed-kubernetes-latest-stable/
```

### deploy custom bundle charmed-kubernetes
```
juju model-config default-series=focal
juju deploy ./bundle.yaml
```

### setting HA k8s using keepalived
```
# deploy keepalived
juju deploy keepalived
# add relation keepalived to kubeapi-lb
juju add-relation keepalived:juju-info kubeapi-load-balancer:juju-info
# configure vip and hostname
export VIP=10.10.10.100
export VIP_HOSTNAME=k8s.btech.com
# add both the new hostname and VIP to the API server certificate
juju config keepalived virtual_ip=$VIP
juju config keepalived vip_hostname=$VIP_HOSTNAME
# Configure kubernetes-control-plane to use the VIP as the advertised Kubernetes API endpoint
juju config kubernetes-control-plane loadbalancer-ips="$VIP"
# precheck
kubectl get pods --all-namespaces
openssl s_client -connect $VIP:443 | openssl x509 -noout -text
```

### Monitoring with Prometheus, Grafana, and Telegraf
```
# deploy prometheus, grafana, and telegraf
juju deploy prometheus2 --to 6
juju deploy grafana --to 6
juju deploy telegraf
# add relation
juju add-relation prometheus2:grafana-source grafana:grafana-source
juju add-relation telegraf:prometheus-client prometheus2:target
juju add-relation kubernetes-control-plane:juju-info telegraf:juju-info
juju add-relation kubernetes-control-plane:prometheus2 prometheus2:manual-jobs
juju add-relation kubernetes-control-plane:grafana grafana:dashboards
juju add-relation prometheus:certificates easyrsa:client
juju add-relation kubernetes-worker:juju-info telegraf:juju-info
juju add-relation kubernetes-worker:scrape prometheus2:scrape
juju add-relation etcd:grafana grafana:dashboards
juju add-relation etcd:prometheus prometheus2:manual-jobs
```

### Ceph
```
juju deploy -n 3 ceph-mon --to lxd:6,lxd:7,lxd:8
juju deploy --config maas-baremetal-k8s-tutorial/ceph-osd.yaml cs:ceph-osd -n 3 --to 6,7,8
# relate ceph-mon and ceph-osd
juju add-relation ceph-mon ceph-osd
# add storage relations
juju add-relation ceph-mon:admin kubernetes-control-plane
juju add-relation ceph-mon:client kubernetes-control-plane
```
