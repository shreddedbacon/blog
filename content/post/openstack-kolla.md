---
title: "Openstack Kolla"
date: 2019-02-03T12:31:27+11:00
---

I recently posted about running Openstack at home, and how I used the RDO project. Well things changed.

<!--more-->
I color-coded my networks to make it easier to look at, and I added a forth node, this one has 16GB RAM in it and is slightly larger running an i7 5500U processor. Seems to be fine.

I also discovered the `Kolla-Ansible` project, and wow. It is much nicer to use and has way more configuration options than packstack.

I started with re-installing each of my nodes with CentOS 7, and configuring their networks. Making sure to have a `bond0` and `bond1` on each node, this makes thing easier later on where `bond0` will be the management/tunnel/storage etc. and `bond1` will be for external networks

# Setup
## Configuration
Using [this](https://docs.openstack.org/kolla-ansible/rocky/user/quickstart.html) guide to get the nodes ready, the next steps involved configuring the `globals.yml` file.
```
yum install epel-release
yum install python-pip
pip install -U pip
yum install python-devel libffi-devel gcc openssl-devel libselinux-python
yum install ansible
pip install kolla-ansible
cp -r /usr/share/kolla-ansible/etc_examples/kolla /etc/
cp /usr/share/kolla-ansible/ansible/inventory/* .
```
Once done, I created a new inventory file based off the multinode deployment `cp multinode stampyhome` only modifying these hosts
```
[control]
control1.stampyhome.com
[network]
control1.stampyhome.com
[inner-compute]
[external-compute]
node1.stampyhome.com
node2.stampyhome.com
node3.stampyhome.com
node4.stampyhome.com
[compute:children]
inner-compute
external-compute
[monitoring]
control1.stampyhome.com
[storage]
control1.stampyhome.com
```

Make the neccessary adjustments as below to suit my environment
```
---
openstack_service_workers: 2
barbican_worker_image: shreddedbacon/barbican-worker
barbican_keystone_listener_image: shreddedbacon/barbican-keystone-listener
barbican_api_image: shreddedbacon/barbican-api
kolla_base_distro: "centos"
kolla_install_type: "binary"
openstack_release: "rocky"
kolla_internal_vip_address: "<external_vip>"
network_interface: "bond0"
neutron_external_interface: "bond1"
neutron_tenant_network_types: "vxlan,vlan"
enable_barbican: "yes"
enable_chrony: "yes"
enable_cinder: "yes"
enable_cinder_backup: "yes"
enable_cinder_backend_nfs: "yes"
enable_fluentd: "yes"
enable_haproxy: "yes"
enable_heat: "no"
enable_horizon: "yes"
enable_horizon_neutron_lbaas: "no"
enable_horizon_octavia: "{{ enable_octavia | bool }}"
enable_neutron_dvr: "yes"
enable_neutron_lbaas: "yes"
enable_nova_ssh: "yes"
enable_octavia: "yes"
glance_enable_rolling_upgrade: "no"
barbican_crypto_plugin: "simple_crypto"
barbican_library_path: "/usr/lib/libCryptoki2_64.so"
ironic_dnsmasq_dhcp_range:
tempest_image_id:
tempest_flavor_ref_id:
tempest_public_network_id:
tempest_floating_network_name:
#octavia_loadbalancer_topology: "ACTIVE_STANDBY"
#octavia_amp_boot_network_list: <octavia_external_network>
#octavia_amp_secgroup_list: <octavia_security_group>
#octavia_amp_flavor_id: octavia
```

As you can see, I am using custom images for barbican, this is because the ones that are in the repository are broken. `python-pyasn` is not new enough. Some dockerfiles, and new builds solves this.
```
#cat barbican-api/Dockerfile
FROM kolla/centos-binary-barbican-api:rocky
USER root
COPY python2-pyasn1-0.3.2-2.fc27.noarch.rpm .
RUN rpm -U python2-pyasn1-0.3.2-2.fc27.noarch.rpm
USER barbican
```
```
#cat barbican-worker/Dockerfile
FROM kolla/centos-binary-barbican-worker:rocky
USER root
COPY python2-pyasn1-0.3.2-2.fc27.noarch.rpm .
RUN rpm -U python2-pyasn1-0.3.2-2.fc27.noarch.rpm
USER barbican
```
```
#cat barbican-keystone-listener/Dockerfile
FROM kolla/centos-binary-barbican-keystone-listener:rocky
USER root
COPY python2-pyasn1-0.3.2-2.fc27.noarch.rpm .
RUN rpm -U python2-pyasn1-0.3.2-2.fc27.noarch.rpm
USER barbican
```
```
docker build -t shreddedbacon/barbican-keystone-listener:rocky barbican-keystone-listener/.
docker build -t shreddedbacon/barbican-worker:rocky barbican-worker/.
docker build -t shreddedbacon/barbican-api:rocky barbican-api/.
```

The sections for Octavia at the bottom need to be commented out on the initial deployment as the items used by them aren't available on an initial deployment.

We also need to generate a bunch of passwords for our deployment
```
kolla-genpwd
```
This will create `/etc/kolla/passwords.yml`, don't lose it!

## Cinder
I am using NFS as my cinder backend, configuration needs to be made to allow this to work.
```
cat < etc/kolla/config/nfs_shares >> EOF
<nfs_server>:/volume1/openstack
EOF
```

## Networking
Since I am using VLAN provider networks, I need to inform neutron about this
```
cat /etc/kolla/config/neutron/ml2_conf.ini
[ml2_type_vlan]
network_vlan_ranges = physnet1:100:500
```

## Octavia
Since we are using Octavia, we also need certificates for the amphora to use. I found [this](https://www.lijiawang.org/posts/kolla-octavia.html) guide really useful for this section

Clone the octavia repository, modify the certifate generation script with the octavia ca password, generate the certificates and put them into the configuration directory. If you don't do this step, prechecks and deploy will fail.
```
git clone https://review.openstack.org/p/openstack/octavia
cd octavia
grep octavia_ca /etc/kolla/passwords.yml
#octavia_ca_password: asdgsdgsdg8sdgsdg88asdg8sadg
sed -i 's/foobar/asdgsdgsdg8sdgsdg88asdg8sadg/g' bin/create_certificates.sh
./bin/create_certificates.sh cert $(pwd)/etc/certificates/openssl.cnf
mkdir -p /etc/kolla/config/octavia
cp cert/{private/cakey.pem,ca_01.pem,client.pem} /etc/kolla/config/octavia/
ls -al /etc/kolla/config/octavia/
```
# Deploy
Once that is all done, run `bootstrap-servers`, `prechecks` and then finaly `deploy`
```
kolla-ansible -i stampyhome bootstrap-servers
kolla-ansible -i stampyhome prechecks
kolla-ansible -i stampyhome deploy
```
After some time, you will have a running Openstack.

## Post Deploy
After deploying, there are some tasks that need to be done

```
pip install python-openstackclient python-glanceclient \
	python-neutronclient python-octaviaclient python-barbicanclient
## Create the admin envars and source them
kolla-ansible post-deploy
. /etc/kolla/admin-openrc.sh
```
### Octavia
Octavia will not be working after the first deploy, as we need some things for it to work
```
openstack flavor create --id octavia --disk 20 \
	--private --ram 512 --vcpus 1 octavia
openstack --os-username octavia \
	--os-password <octavia_keystone_password> \
	keypair create --public-key /root/.ssh/id_rsa.pub octavia_ssh_key
openstack security group create \
	--description 'used by Octavia amphora instance' octavia
openstack security group rule create \
	--protocol icmp 188b91d4-8bc9-4a31-83c4-c37a2ec400ff
openstack security group rule create \
	--protocol tcp --dst-port 5555 \
	--egress 188b91d4-8bc9-4a31-83c4-c37a2ec400ff
openstack security group rule create \
	--protocol tcp --dst-port 9443 \
	--ingress 188b91d4-8bc9-4a31-83c4-c37a2ec400ff
```
### Amphora Image
Some additional things are required to get amphora to build

```
pip install diskimage-builder
cd octavia
./diskimage-create/diskimage-create.sh -i centos -s 5
openstack image create --container-format bare \
	--disk-format qcow2 \
	--private --file /root/octavia/amphora-x64-haproxy.qcow2 \
	--tag amphora amphora
```
### Octavia External Network
Since I am using VLAN provider networks, I picked one that is designed for the openstack network in general
```
openstack network create --share --external \
	--provider-physical-network physnet1 \
	--provider-segment 400 --provider-network-type vlan extnet4
openstack subnet create --no-dhcp \
	--allocation-pool start=10.1.4.50,end=10.1.4.150 \
	--network extnet4 --subnet-range 10.1.4.0/24 \
	--gateway 10.1.4.254 extnet4-subnet
```
With all of these, I need to take the network ID, and the security group ID and populate the `octavia_amp_boot_network_list` and `octavia_amp_secgroup_list`, uncomment the 4 octavia items and run a deploy for just Octavia.
```
kolla-ansible -i stampyhome deploy --tags octavia
```
Again, after some time, octavia will be done and you can deploy load balancers.

# Issues
## Octavia / Barbican
I can't create `TERMINATED_HTTPS` load balancers from the dashboard, either this is still an in-development feature, or its genuinely broken.
I was sort of able to get one to work via the CLI tools, but this was cumbersome as it required using secrets, acls and secret containers and I gave up trying for a bit.

I'll give HTTPS pass through a go at some stage though.
