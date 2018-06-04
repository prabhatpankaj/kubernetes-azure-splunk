### Create a default Resource Group in a location

The guide assumes you've installed the [Azure CLI 2.0](https://github.com/azure/azure-cli#installation), and will be creating resources in the `westus2` location, within a resource group named `kubernetes`. To create this resource group, simply run the following command:

```shell
az group create -n kubernetes -l westus2
```
### Virtual Network

In this section a dedicated [Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) (VNet) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-vnet` custom VNet network with a subnet `kubernetes` provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.:

```shell
az network vnet create -g kubernetes \
  -n kubernetes-vnet \
  --address-prefix 10.240.0.0/24 \
  --subnet-name kubernetes-subnet
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall ([Network Security Group](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-nsg)) and assign it to the subnet:

```shell
az network nsg create -g kubernetes -n kubernetes-nsg
```

```shell
az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --network-security-group kubernetes-nsg
```

Create a firewall rule that allows external SSH and HTTPS:

```shell
az network nsg rule create -g kubernetes \
  -n kubernetes-allow-ssh \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 22 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1000
```

```shell
az network nsg rule create -g kubernetes \
  -n kubernetes-allow-api-server \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 6443 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1001
```

> An [external load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-vnet` VNet network:

```shell
az network nsg rule list -g kubernetes --nsg-name kubernetes-nsg --query "[].{Name:name, \
  Direction:direction, Priority:priority, Port:destinationPortRange}" -o table
```

> output

```shell
Name                         Direction      Priority    Port
---------------------------  -----------  ----------  ------
kubernetes-allow-ssh         Inbound            1000      22
kubernetes-allow-api-server  Inbound            1001    6443
```
## Virtual Machines

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the [cri-containerd container runtime](https://github.com/kubernetes-incubator/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Masters

Create three compute instances which will host the Kubernetes control plane in `controller-as` [Availability Set](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/regions-and-availability#availability-sets):

```shell
az vm availability-set create -g kubernetes -n master-as
```

```shell
for i in 0; do
    echo "[Master ${i}] Creating public IP..."
    az network public-ip create -n master-${i}-pip -g kubernetes > /dev/null

    echo "[Master ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n controller-${i}-nic \
        --private-ip-address 10.240.0.1${i} \
        --public-ip-address master-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet 

    echo "[Master ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n master-${i} \
        --image Canonical:UbuntuServer:16.04.0-LTS:latest \
        --generate-ssh-keys \
        --nics master-${i}-nic \
        --availability-set master-as \
        --nsg '' > /dev/null
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.240.0.0/16`, which supports 254 subnets.

Create one compute instances which will host the Kubernetes worker nodes in `worker-as` Availability Set:

```shell
az vm availability-set create -g kubernetes -n worker-as
```

```shell
for i in 0; do
    echo "[Worker ${i}] Creating public IP..."
    az network public-ip create -n worker-${i}-pip -g kubernetes > /dev/null

    echo "[Worker ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n worker-${i}-nic \
        --private-ip-address 10.240.0.2${i} \
        --public-ip-address worker-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding > /dev/null

    echo "[Worker ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n worker-${i} \
        --image Canonical:UbuntuServer:16.04.0-LTS:latest \
        --nics worker-${i}-nic \
        --tags pod-cidr=10.200.${i}.0/24 \
        --availability-set worker-as \
        --nsg '' > /dev/null
done
```

### Verification

List the compute instances in your default compute zone:

```shell
az vm list -d -g kubernetes -o table
```

> output

```shell
Name          ResourceGroup    PowerState    PublicIps       Location
------------  ---------------  ------------  --------------  ----------
master-0      kubernetes       VM running    XX.XXX.XXX.XXX  westus2
worker-0      kubernetes       VM running    XX.XXX.XXX.XXX  westus2
```
