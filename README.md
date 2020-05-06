# 5GIIK-testbed

This GitHub repository includes the main procedures to deploy 5GIIK testbed for Network Slicing and Orchestration.
5GIIK is an academic platform has been created in the information security department of NTNU university in Norway. The main target is to deploy a small-scale platform that helps academic purposes for realizing network slicing and service orchestration. 5GIIK emulates the Radio Access Network (RAN) by using Software Radio Systems (SRS) LTE and the Core Network (CN) by utilizing OpenAirInterface (OAI) EPC or NetxtEPC. 5GIIK benefits from a service orchestrator (Open-Source MANO (OSM)), Virtualized Infrastructure Managers (VIMs) in our case OpenStack platforms, and SDN controllers for various network domains (5G-EmPOWER and ONOS). All required procedures for deploying the 5GIIK platform are explained in the following.


# Service Orchestrator Deployment (Open-Source Mano (OSM))

You need to have an Ubuntu18.04 (64-bit variant required) system with 2 CPUs, 8 GB RAM, at least 40GB disk, and a single interface with Internet access for deploying OSM release 7. It is recommended to install the OSM with debugging capabilities in order to trace back the possible errors that may occur.

```
wget https://osm-download.etsi.org/ftp/osm-7.0-seven/install_osm.sh
chmod +x install_osm.sh
./install_osm.sh 2>&1 | tee osm_install_log.txt
```

You can then check if the installation has been done correctly with the following commands. These commands show the created docker containers that are built on your machine.

```
docker stack ps osm |grep -i running
docker service ls
```

Then you can simply navigate to the OSM GUI by accessing the IP address of your host machine and using "admin/admin" as OSM credentials. It is recommended to change the password after the first time login. For accessing the OSM installed on a remote machine, if you did not install ssh server, execute the following command on your main host and then you will be able to access the host machine remotely.

```
sudo apt-get install openssh-server
service ssh status
```

# Virtualized Infrastructure Manager (VIM) deployment (OpenStack)

Usually deploying OpenStack is not straight forward as it seems in the first step. You need to deal with multiple problems. It is strongly recommended to use a server with high capacity in terms of RAM and CPUs in order to avoid possible further problems. One of the possible procedures to deploy OpenStack is to install all-in-one implementations. In the following, the procedures are explained. First prepare a server with the minimum requirements such as at least 64GB of RAM, 100GB of storage, and 7 CPUs. You need to install the Minimal version of CentOS7 Linux distribution in order to be able to run OpenStack on top of it.

After installation you execute following commands on your CentOS7 server in order to update your system:

```
yum install epel-release
yum update
```

The same as the OSM, for accessing the CentOS7 server remotely, simply execute the following commands on your server:

```
sudo yum –y install openssh-server openssh-clients
sudo systemctl status sshd
```

Now your CentOS7 server is ready for OpenStack installation. RDO project is one possible solution for deploying all-in-one OpenStack. However, you should modify some procedures for avoiding possible errors that may occur during installation.

First of all, execute the following commands to turn off NetworkManager and Firewall on your server and enable normal Networking:

```
sudo systemctl disable firewalld NetworkManager
sudo systemctl enable network
```

For development purposes, it is recommended to disable SELINUX on your server.

```
sed -i s/^SELINUX=.*$/SELINUX=permissive/ /etc/selinux/config
```

Then if your environment is a non-English locale, change it by adding the following in your /etc/environment.

```
LANG=en_US.utf-8
LC_ALL=en_US.utf-8
```

At this point, it is better to reboot your server.
After your server is relaunched, you are able to start the OpenStack installation. The current and stable version of the RDO solution for OpenStack installation is stein. So, execute the following commands in order to install the required repositories on your server.

```
sudo yum update -y
sudo yum install -y centos-release-openstack-stein
sudo yum update -y
sudo yum install -y openstack-packstack
```

In this stage, you need to create a configuration file called the answer file which includes all the services that can be provided by the OpenStack. Creating an answer file helps you to customize the services that you prefer to have on your OpenStack platform. You can modify this file and then execute it to install OpenStack all-in-one solution. So, first, execute the following command to create the answer file.

```
packstack --gen-answer-file=answer.txt
```

Then open this answer file with your favorite text editor and customize the services you would prefer to run on your OpenStack. But it is necessary to map your ethernet interface on the external bridge (br-ex) in order to have network connectivity to outside of your OpenStack environment. So, apart from your changes in the answer file, do the following modifications.

```
--os-neutron-ovs-bridge-mappings=extnet:br-ex 
--os-neutron-ovs-bridge-interfaces=br-ex:<your ethernet interface name> 
--os-neutron-ml2-type-drivers=vxlan,flat
```

Your server is ready to install OpenStack now. So, execute this command to deploy OpenStack on your server.

```
packstack --answer-file=answer.txt
```

If you get an error regarding Puppet installation, first execute the following command and then redo the previous step.

```
yum downgrade leatherman
```

After the installation is done, Then you should map the ethernet interface on external bridge on your server as you configured before in OpenStack installation. So create /etc/sysconfig/network-scripts/ifcfg-br-ex and paste the following content in it.

```
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=<your ethernet interface IP Address>
NETMASK=<your network mask>
GATEWAY=<the gateway of your network>
DNS=<the DNS server of your network>
ONBOOT=yes
```

And on your ethernet interface in /etc/sysconfig/network-scripts/ifcfg-<your ethernet interface name>, paste the following content.

```
DEVICE=<your ethernet interface name>
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
```

Finally, reboot your server or execute the following command.

```
service network restart
```

At this point, you should be able to access the OpenStack dashboard by using its credential information located in the keystonerc_admin file.

You need to configure external and internal networks, possible routers, firewall, security groups on your OpenStack platform. You should also upload the required images you want to use for deploying network services on OpenStack. Usually, cloud-based Ubuntu images are the ones that are required for deploying network services such as RAN, CN instances of an LTE network. So, one simple approach is by using the OpenStack dashboard to create an image, and the other is by using CLI. To use CLI first of all source the keystonerc_admin file and then execute the following commands to download the required image and then import it on the OpenStack dashboard.
For Ubuntu 16.04 on your server execute the following.

```
wget https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
openstack image create --file="./xenial-server-cloudimg-amd64-disk1.img" --container-format=bare --disk-format=qcow2 ubuntu1604
```

For Ubuntu 18.04 on your server execute the following.

```
wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
openstack image create --file="./bionic-server-cloudimg-amd64.img" --container-format=bare --disk-format=qcow2 ubuntu1804
```

At this point, you can integrate your VIM account (OpenStack) to the service orchestrator (OSM). On your OSM machine execute the following.

```
osm vim-create --name <a name for your VIM account> --user <admin or your tenant username> --password <the required password for your tenant account> --auth_url <url of your OpenStack dashboard>:5000/v3/ --tenant <admin or your related tanant name> --account_type openstack --config='{<possible additional configuration on your OpenStack platform>}'
```

# Deploying Network Services

At this stage, we have created our service orchestrator which is now integrated with one VIM account. For deploying network services, we need to create Virtual Network Function Descriptors (VNFDs) and Network Service Descriptors (NSDs). For creating network slices, some launched instances (services) on the VIM platform are needed to be chained to create one/several network slice(s). In this particular case, OAI EPC/NextEPC and srsLTE RAN are the two VNFs that emulate CN and RAN respectively. Clone the required descriptors from this repository and upload them to the OSM dashboard in order to instantiate network services.

# Creating base images for service instantiation

As mentioned before, you should use cloud-based ubuntu images in order to install srsLTE for RAN and OAI EPC/NextEPC for CN. In the following the procedures for installing these base images are explained.

### OAI EPC installation on ubuntu 

You should install low latency kernel on Ubuntu 18.04 as a base image for OAI EPC.

```
sudo apt-get install linux-image-lowlatency linux-headers-lowlatency
```

After that you should install Cassandra which acts as HSS database for OAI EPC.

```
echo "deb http://www.apache.org/dist/cassandra/debian 21x main" | $SUDO  tee -a /etc/apt/sources.list.d/cassandra.sources.list
curl https://downloads.apache.org/cassandra/KEYS | $SUDO  apt-key add –
$SUDO $INSTALLER install $OPTION curl openjdk-8-jre

```

Then you should reboot the instance in order to make the changes. Then, you should clone OAI from its main repository.

```
git clone https://github.com/OPENAIRINTERFACE/openair-cn.git
cd openair-cn/scripts
git checkout develop
cd ..
```
```
git clone https://github.com/OPENAIRINTERFACE/openair-cn-cups.git
cd openair-cn-cups/                  
git checkout develop
```
#### OAI EPC - HSS 
First go to the following directory and execute these commands one after another.

```
cd openair-cn/scripts
./build_cassandra --check-installed-software --force
sudo service cassandra stop
sudo update-alternatives --config java
sudo service cassandra start
sudo service cassandra stop
sudo rm -rf /var/lib/cassandra/data/system/*
sudo rm -rf /var/lib/cassandra/commitlog/*
sudo rm -rf /var/lib/cassandra/data/system_traces/*
sudo rm -rf /var/lib/cassandra/saved_caches/*
sudo service cassandra start
./build_hss_rel14 --check-installed-software --force
./build_hss_rel14 --clean
Cassandra_Server_IP='127.0.0.1'
MY_REALM='srslte.com'
cqlsh --file ../src/hss_rel14/db/oai_db.cql $Cassandra_Server_IP
./data_provisioning_users --apn default.srslte.com --apn2 internet --key 6FD60656E2B7F86FBEE9A31518AC18BE --imsi-first 901700000011201 --msisdn-first 001010000011201 --mme-identity mme.$MY_REALM --no-of-users 20 --realm $MY_REALM --truncate True  --verbose True --cassandra-cluster $Cassandra_Server_IP

./data_provisioning_mme --id 3 --mme-identity mme.$MY_REALM --realm $MY_REALM --ue-reachability 1 --truncate True  --verbose True -C $Cassandra_Server_IP

openssl rand -out $HOME/.rnd 128
PREFIX='/usr/local/etc/oai'
sudo mkdir -m 0777 -p $PREFIX
sudo mkdir -m 0777 -p $PREFIX/freeDiameter
sudo mkdir -m 0777 -p $PREFIX/logs
sudo mkdir -m 0777 -p logs
sudo chmod 777 $PREFIX
sudo chmod 777 $PREFIX/freeDiameter
sudo chmod 777 $PREFIX/logs
sudo chmod 777 logs
sudo cp ../etc/acl.conf ../etc/hss_rel14_fd.conf $PREFIX/freeDiameter
sudo cp ../etc/hss_rel14.conf ../etc/hss_rel14.json $PREFIX
sudo cp ../etc/oss.json $PREFIX
declare -A HSS_CONF
HSS_CONF[@PREFIX@]=$PREFIX
HSS_CONF[@REALM@]=$MY_REALM
HSS_CONF[@HSS_FQDN@]="hss.${HSS_CONF[@REALM@]}"
HSS_CONF[@cassandra_Server_IP@]=$Cassandra_Server_IP
HSS_CONF[@OP_KEY@]='AF8C8C9B229BFADC722A74A3FBF3A490'
HSS_CONF[@ROAMING_ALLOWED@]='true'
for K in "${!HSS_CONF[@]}"; do  egrep -lRZ "$K" $PREFIX | xargs -0 -l sed -i -e "s|$K|${HSS_CONF[$K]}|g"; done
../src/hss_rel14/bin/make_certs.sh hss ${HSS_CONF[@REALM@]} $PREFIX
sudo sed -i -e 's/#ListenOn/ListenOn/g' $PREFIX/freeDiameter/hss_rel14_fd.conf
oai_hss -j $PREFIX/hss_rel14.json –onlyloadkey
```

#### OAI EPC - MME

Remain in the same directory and follow up these commands to install MME. Notice to change MCC and MNC according to your SIMCard information or program the SIMCard with your favorite values.

```
./build_mme --check-installed-software --force
./build_mme --clean
virsh net-list --all
openssl rand -out $HOME/.rnd 128
INSTANCE=1
PREFIX='/usr/local/etc/oai'
cp ../etc/mme_fd.sprint.conf  $PREFIX/freeDiameter/mme_fd.conf
cp ../etc/mme.conf  $PREFIX
declare -A MME_CONF
MME_CONF[@MME_S6A_IP_ADDR@]="127.0.0.11"
MME_CONF[@INSTANCE@]=$INSTANCE
MME_CONF[@PREFIX@]=$PREFIX
MME_CONF[@REALM@]='srslte.com'
MME_CONF[@PID_DIRECTORY@]='/var/run'
MME_CONF[@MME_FQDN@]="mme.${MME_CONF[@REALM@]}"
MME_CONF[@HSS_HOSTNAME@]='hss'
MME_CONF[@HSS_FQDN@]="${MME_CONF[@HSS_HOSTNAME@]}.${MME_CONF[@REALM@]}"
MME_CONF[@HSS_IP_ADDR@]='127.0.0.1'
MME_CONF[@MCC@]='<your MCC of your SIMCard>'
MME_CONF[@MNC@]='<your MNC of your SIMCard>'
MME_CONF[@MME_GID@]='4'
MME_CONF[@MME_CODE@]='1'
MME_CONF[@TAC_0@]='600'
MME_CONF[@TAC_1@]='601'
MME_CONF[@TAC_2@]='602'
MME_CONF[@MME_INTERFACE_NAME_FOR_S1_MME@]='ens3:m1c'
MME_CONF[@MME_IPV4_ADDRESS_FOR_S1_MME@]='192.168.247.102/24'
MME_CONF[@MME_INTERFACE_NAME_FOR_S11@]='ens3:m11'
MME_CONF[@MME_IPV4_ADDRESS_FOR_S11@]='172.16.1.102/24'
MME_CONF[@MME_INTERFACE_NAME_FOR_S10@]='ens3:m10'
MME_CONF[@MME_IPV4_ADDRESS_FOR_S10@]='192.168.10.110/24'
MME_CONF[@OUTPUT@]='CONSOLE'
MME_CONF[@SGW_IPV4_ADDRESS_FOR_S11_TEST_0@]='172.16.1.104/24'
MME_CONF[@SGW_IPV4_ADDRESS_FOR_S11_0@]='172.16.1.104/24'
MME_CONF[@PEER_MME_IPV4_ADDRESS_FOR_S10_0@]='0.0.0.0/24'
MME_CONF[@PEER_MME_IPV4_ADDRESS_FOR_S10_1@]='0.0.0.0/24'
TAC_SGW_TEST='7'
tmph=`echo "$TAC_SGW_TEST / 256" | bc`
tmpl=`echo "$TAC_SGW_TEST % 256" | bc`
MME_CONF[@TAC-LB_SGW_TEST_0@]=`printf "%02x\n" $tmpl`
MME_CONF[@TAC-HB_SGW_TEST_0@]=`printf "%02x\n" $tmph`
MME_CONF[@MCC_SGW_0@]=${MME_CONF[@MCC@]}
MME_CONF[@MNC3_SGW_0@]=`printf "%03d\n" $(echo ${MME_CONF[@MNC@]} | sed 's/^0*//')`
TAC_SGW_0='600'
tmph=`echo "$TAC_SGW_0 / 256" | bc`
tmpl=`echo "$TAC_SGW_0 % 256" | bc`
MME_CONF[@TAC-LB_SGW_0@]=`printf "%02x\n" $tmpl`
MME_CONF[@TAC-HB_SGW_0@]=`printf "%02x\n" $tmph`
MME_CONF[@MCC_MME_0@]=${MME_CONF[@MCC@]}
MME_CONF[@MNC3_MME_0@]=`printf "%03d\n" $(echo ${MME_CONF[@MNC@]} | sed 's/^0*//')`
TAC_MME_0='601'
tmph=`echo "$TAC_MME_0 / 256" | bc`
tmpl=`echo "$TAC_MME_0 % 256" | bc`
MME_CONF[@TAC-LB_MME_0@]=`printf "%02x\n" $tmpl`
MME_CONF[@TAC-HB_MME_0@]=`printf "%02x\n" $tmph`
MME_CONF[@MCC_MME_1@]=${MME_CONF[@MCC@]}
MME_CONF[@MNC3_MME_1@]=`printf "%03d\n" $(echo ${MME_CONF[@MNC@]} | sed 's/^0*//')`
TAC_MME_1='602'
tmph=`echo "$TAC_MME_1 / 256" | bc`
tmpl=`echo "$TAC_MME_1 % 256" | bc`
MME_CONF[@TAC-LB_MME_1@]=`printf "%02x\n" $tmpl`
MME_CONF[@TAC-HB_MME_1@]=`printf "%02x\n" $tmph`

for K in "${!MME_CONF[@]}"; do    egrep -lRZ "$K" $PREFIX | xargs -0 -l sed -i -e "s|$K|${MME_CONF[$K]}|g";   ret=$?;[[ ret -ne 0 ]] && echo "Tried to replace $K with ${MME_CONF[$K]}"; done

sudo ./check_mme_s6a_certificate $PREFIX/freeDiameter mme.${MME_CONF[@REALM@]}

```





















































