# OKD_Installation
OKD INSTALLATION  </br>
                                        --BY PRACHI UPADHYAY </br>
====================================================================== </br>
### Required machines </br>
    The smallest OpenShift Container Platform clusters require the following hosts: 
    One temporary bootstrap machine </br>
    Three control plane, or master machines </br>
    At least two worker machines. </br>

#### *NOTE: 
        The cluster requires the bootstrap machine to deploy the OpenShift Container Platform cluster on the three control plane machines. You can remove the bootstrap machine after you install the cluster.

### Network connectivity requirements
      During the initial boot, the machines require either a DHCP server or that static IP addresses be set in order to establish a network connection to  download their Ignition config files. Additionally, each OpenShift Container Platform node in the cluster must have access to a Network Time Protocol    (NTP) server. If a DHCP server provides NTP servers information, the chrony time service on the Red Hat Enterprise Linux CoreOS (RHCOS) machines read     the information and can sync the clock with the NTP servers.

### Minimum resource requirements
     | Machine | No. of machine | Memory [GB] | Storage[Gib] | vCPU | OS | NIC card | required Bandwidth of N/W in [Gbps] | DNS |
     ----------------------------------------------------------------------------------------------------------------------------
     | 1 | 30 | 2 | Fedora/Centos/Rhel | 1 | 10 | HAproxy | 1 |
      2
      30
2
Fedora/Centos/Rhel
1
10
Bastion
1
2
30
2
Fedora/Centos/Rhel
1
10
Bootstrap
1
16
100
4
CoreOS
1
10
Master
3
16
100
4
CoreOS
1
10
Worker
3
16
100
4
CoreOS
1
10


Prerequisites

Configure DHCP or set static IP addresses on each node.


Provision the required load balancers. (HA-Proxy)


Configure the ports for your machines.


Configure DNS.


NTP configuration


INVENTORY OF OKD:

SR.no.
Server
Role
Hostname
IP Address
1
Bastion
DNS/HAproxy
bastion.okd4.keenable.com
192.168.1.50
2
Bootstrap


bootstrap.okd4.keenable.com
192.168.1.51
3
okd-Master0
okd4-VM
master0.okd4.keenable.com
192.168.1.52
4
okd-Master1
okd4-VM
master1.okd4.keenable.com
192.168.1.53
5
okd-Master2
okd4-VM
master2.okd4.keenable.com
192.168.1.54
6
okd-Worker0
okd4-VM
worker0.okd4.keenable.com
192.168.1.55
7
okd-Worker1
okd4-VM
worker1.okd4.keenable.com
192.168.1.56


Pre-installation task:

1.Configure DNS
2.Configure haproxy

1.Configure DNS 

# cat /etc/named.conf      (to create zones)
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
    listen-on port 53 { 127.0.0.1; 192.168.1.50; };
    listen-on-v6 port 53 { ::1; };
    directory     "/var/named";
    dump-file     "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file    "/var/named/data/named.secroots";
    recursing-file    "/var/named/data/named.recursing";
    	allow-query 	{ localhost;  192.168.1.0/24;};
//    	forwarders {
  //          	192.168.1.1;
	//	};
  	//  forward only;

    /*
     - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
     - If you are building a RECURSIVE (caching) DNS server, you need to enable
   	recursion.
     - If your recursive DNS server has a public IP address, you MUST enable access
   	control to limit queries to your legitimate users. Failing to do so will
   	cause your server to become part of large scale DNS amplification
   	attacks. Implementing BCP38 within your network would greatly
   	reduce such attack surface
    */
    recursion yes;

    dnssec-validation yes;

    managed-keys-directory "/var/named/dynamic";
    geoip-directory "/usr/share/GeoIP";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";

    /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
    include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
    	channel default_debug {
            	file "data/named.run";
            	severity dynamic;
    	};
};

zone "." IN {
    type hint;
    file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

zone "keenable.com" IN {
type master;
file "forward.com";
allow-update { none; };
};
zone "1.168.192.in-addr.arpa" IN {
type master;
file "reverse.com";
allow-update { none; };
};

forward .com
# cat /var/named/forward.com
$TTL 1W
@    IN    SOA    ns1.keenable.com.    root (
   		 2019070700    ; serial
   		 3H   	 ; refresh (3 hours)
   		 30M   	 ; retry (30 minutes)
   		 2W   	 ; expiry (2 weeks)
   		 1W )   	 ; minimum (1 week)
    IN    NS    ns1.keenable.com.
    IN    MX 10    smtp.keenable.com.
;
;
ns1.keenable.com.   	 IN    A    192.168.1.50
smtp.keenable.com.   	 IN    A    192.168.150
;
haproxy.keenable.com.   	 IN    A    192.168.1.5
haproxy.okd4.keenable.com.    IN    A    192.168.1.5
;
api.okd4.keenable.com.   	 IN    A    192.168.1.50
api-int.okd4.keenable.com.    IN    A    192.168.1.50
;
*.apps.okd4.keenable.com.    IN    A    192.168.1.50
;
bootstrap.okd4.keenable.com.    IN    A    192.168.1.51
;
master0.okd4.keenable.com.    IN    A    192.168.1.52
master1.okd4.keenable.com.    IN    A    192.168.1.53
master2.okd4.keenable.com.    IN    A    192.168.1.54
;
worker0.okd4.keenable.com.    IN    A    192.168.1.55
worker1.okd4.keenable.com.    IN    A    192.168.1.56
;
;EOF

reverse.com
 # cat /var/named/reverse.com
$TTL 1W
@   IN  SOA 	ns1.keenable.com.   root (
2019070700 ; serial
3H ; refresh (3 hours)
30M ; retry (30 minutes)
2W ; expiry (2 weeks)
1W ) ; minimum (1 week)
   	IN  NS      	ns1.keenable.com.

ns1       	IN  A   192.168.1.50

50	IN  PTR     	ns1.keenable.com.
51 	IN  PTR    	bootstrap.okd4.keenable.com.
52	IN  PTR     	master0.okd4.keenable.com.
53	IN  PTR     	master1.okd4.keenable.com.
54	IN  PTR     	master2.okd4.keenable.com.
55	IN  PTR     	worker0.okd4.keenable.com.
56	IN  PTR     	worker1.okd4.keenable.com.
50         IN  PTR             api.okd4.keenable.com.
50         IN  PTR             api-int.okd4.keenable.com.

verify the named.conf, forward zone & reverse zone file with the following command:
$ named-checkconf /etc/named.conf
$ named-checkzone forward.com /var/named/forward.com
$ named-checkzone reverse.com /var/named/reverse.com

$ firewall-cmd --permanent --add-port=53/udp
$ firewall-cmd --reload
$ nslookup ns1.keenable.com      (nslookup <domain.name.com> )
$ nslookup 192.168.1.50              (nslookup <ip_address>

2.Configure haproxy

# cat /etc/haproxy/haproxy.cfg

global
  log     	127.0.0.1 local2
  pidfile 	/var/run/haproxy.pid
  maxconn 	4000
  daemon
defaults
  mode                	http
  log                 	global
  option              	dontlognull
  option http-server-close
  option              	redispatch
  retries             	3
  timeout http-request	10s
  timeout queue       	1m
  timeout connect     	10s
  timeout client      	1m
  timeout server      	1m
  timeout http-keep-alive 10s
  timeout check       	10s
  maxconn             	3000
frontend stats
  bind *:1936
  mode        	http
  log         	global
  maxconn 10
  stats enable
  stats hide-version
  stats refresh 30s
  stats show-node
  stats show-desc Stats for ocp4 cluster
  stats auth admin:redhat
  stats uri /stats
listen api-server-6443
  bind *:6443
  mode tcp
#  server bootstrap bootstrap.okd4.keenable.com:6443 check inter 1s backup
  server master0 master0.okd4.keenable.com:6443 check inter 1s
  server master1 master1.okd4.keenable.com:6443 check inter 1s
  server master2 master2.okd4.keenable.com:6443 check inter 1s
listen machine-config-server-22623
  bind *:22623
  mode tcp
#  server bootstrap bootstrap.okd4.keenable.com:22623 check inter 1s backup
  server master0 master0.okd4.keenable.com:22623 check inter 1s
  server master1 master1.okd4.keenable.com:22623 check inter 1s
  server master2 master2.okd4.keenable.com:22623 check inter 1s
listen ingress-router-443
  bind *:443
  mode tcp
  balance source
  server worker0 worker0.okd4.keenable.com:443 check inter 1s
  server worker1 worker1.okd4.keenable.com:443 check inter 1s
listen ingress-router-80
  bind *:80
  mode tcp
  balance source
  server worker0 worker0.okd4.keenable.com:80 check inter 1s
  server worker1 worker1.okd4.keenable.com:80 check inter 1s
Generating an SSH private key and adding it to the agent:
$ ssh-keygen -t ed25519 -N '' \  -f <path>/<file_name>
$ mkdir OKD
$ ssh-keygen -t ed25519 -N '' -f  OKD/okdkey

Start the ssh-agent process as a background task:
$ eval "$(ssh-agent -s)"

Add your SSH private key to the ssh-agent:
$ ssh-add <path>/<file_name>
$ ssh-add  OKD/okdkey

Download Openshift-installer Program:
$ wget https://github.com/openshift/okd/releases/download/4.10.0-0.okd-2022-07-09-073606/openshift-install-linux-4.10.0-0.okd-2022-07-09-073606.tar.gz

$ tar xvf openshift-install-linux-4.10.0-0.okd-2022-07-09-073606.tar.gz

Installing the OpenShift CLI (oc) by downloading the binary
$ cd OKD/

$ wget https://github.com/openshift/okd/releases/download/4.10.0-0.okd-2022-07-09-073606/openshift-client-linux-4.10.0-0.okd-2022-07-09-073606.tar.gz

$ tar xvf openshift-client-linux-4.10.0-0.okd-2022-07-09-073606.tar.gz

$ oc version

Manually creating the installation configuration file
$ mkdir installation
$ vim install-config.yaml 
apiVersion: v1
baseDomain: keenable.com 
compute: 
- hyperthreading: Enabled 
  name: worker
  replicas: 0 
controlPlane: 
  hyperthreading: Enabled 
  name: master
  replicas: 3 
metadata:
  name: test 
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths": ...}' 
sshKey: 'ssh-ed25519 AAAA...'


*NOTE: 


You must name this configuration file install-config.yaml.

$ cat okdkey.pub
$ mkdir /OKD/installation
$ cp install-config.yaml /OKD/installation/
$ ./openshift-install create manifests --dir installation/
$ cd OKD/installation/manifest/
$ vim cluster-scheduler-02-cofig.yml (change true to false)
$ ./openshift-install create ignition-configs --dir OKD/installation/
$ cp OKD/installation/*.ign /var/www/html/
$ chmod -R 777 /var/www/html/
$ cd /var/www/html/
$ vim b.sh
sudo coreos-installer install --copy-network /dev/sda --ignition-url http://192.168.1.50:80/bootstrap.ign --insecure-ignition 
$ vim m.sh
sudo coreos-installer install --copy-network /dev/sda --ignition-url http://192.168.1.50:80/master.ign --insecure-ignition 
$ vim w.sh
sudo coreos-installer install --copy-network /dev/sda --ignition-url http://192.168.1.50:80/worker.ign --insecure-ignition 
$ systemctl start httpd
$ systemctl enable httpd
$ curl  http://192.168.1.50:80/worker.ign   (to test)

Create 1 Bootstrap, 3 master & 3 worker nodes:
On Bootstrap machine:
$ nmtui            (configure static IP & set hostname)
$ curl http://192.168.1.50:80 b.sh > b.sh

On 3 master machine:
$ nmtui            (configure static IP & set hostname)
$ curl http://192.168.1.50:80 m.sh > m.sh

On 3 worker machine:
$ nmtui            (configure static IP & set hostname)
$ curl http://192.168.1.50:80 w.sh > w.sh

================================================================================================
On Bastion Node:
$ ./openshift-install --dir installation wait-for bootstrap-complete \ 
    --log-level=info
$ export KUBECONFIG=installation/auth/kubeconfig
$ oc get nodes
$ oc get csr
$ for i in `oc get csr --no-headers | grep -i pending |  awk '{ print $1 }'`; do oc adm certificate approve $i; done



