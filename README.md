# OKD 4.5 Bare Metal Install - User Provisioned Infrastructure (UPI)

- [OKD 4.5 Bare Metal Install - User Provisioned Infrastructure (UPI)]
  - [Prepare the 'Bare Metal' environment](#prepare-the-bare-metal-environment)
  - [Download Software](#download-required-software)
  - [Configure Environmental Services](#configure-environmental-services)
  - [Generate and host install files](#generate-and-host-install-files)
  - [Deploy OpenShift](#deploy-openshift)
  - [Monitor the Bootstrap Process](#monitor-the-bootstrap-process)
  - [Remove the Bootstrap Node](#remove-the-bootstrap-node)
  - [Wait for installation to complete](#wait-for-installation-to-complete)
  - [Join Worker Nodes](#join-worker-nodes)
  - [Configure storage for the Image Registry](#configure-storage-for-the-image-registry)
  - [Create the first Admin user](#create-the-first-admin-user)
  - [Access the OpenShift Console](#access-the-openshift-console)
  - [Troubleshooting](#troubleshooting)


## Prepare the 'Bare Metal' environment

> KVM Host is used in this solution.

1. Required machines:
1. One temporary bootstrap machine
   - Name: okd-boostrap
   - 4vcpu
   - 8GB RAM
   - 100GB HDD
   - NIC connected to the OKD network
1. 3 Control Plane virtual machines with minimum settings:
   - Name: okd-cp-# (Example okd-cp-1)
   - 4vcpu
   - 16GB RAM
   - 120GB HDD
   - NIC connected to the OKD network
1. 2 Worker virtual machines (or more if you want) with minimum settings:
   - Name: okd-w-# (Example okd-w-1)
   - 4vcpu
   - 8GB RAM
   - 120GB HDD
   - NIC connected to the OKD network
1. Create a Services virtual machine with minimum settings:
   - Name: okd-svc
   - 4vcpu
   - 4GB RAM
   - 120GB HDD
   - NIC2 connected to the OKD network
1. Boot all virtual machines so they each are assigned a MAC address
1. Shut down all virtual machines except for 'okd-svc'
1. Use the ``` virsh domiflist <domain-name>``` command to record the MAC address of each vm, these will be used later to set static IPs.

## Download Required Software

1. Download [FCOS images from the](https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable)
1. Select 'ISO Image' from the page.
1. Download the following file:
     - fedora-coreos-xx.xxxxx.xx-live.x86_64.iso
1. Download installer [openshift-installer](https://github.com/openshift/okd/releases)
1. Download the following files:
     - openshift-client-linux-4.5.x-x.okd-xxxx-xx-xx-xxxxxx.tar.gz
     - openshift-install-linux-4.5.x-x.okd-xxxx-xx-xx-xxxxxx.tar.gz

## Configure OKD Network on KVM Host
1. Create OKD Network

``` 
[root@sm-epyc-centos8 ~]# vim /etc/libvirt/qemu/networks/okd45.xml 
<network>
  <name>okd45</name>
  <forward mode='nat'/>
  <domain name='okd45.smcloud.local'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
  </ip>
</network> 
```
```
[root@sm-epyc-centos8 ~]# virsh define /etc/libvirt/qemu/networks/okd45.xml
```
```
[root@sm-epyc-centos8 ~]# virsh net-autostart --network okd45
```
```
[root@sm-epyc-centos8 ~]# virsh net-list
 Name               State    Autostart   Persistent
-----------------------------------------------------
 default            active   yes         yes
 okd45              active   yes         yes
```

## Configure Environmental Services

1. Install CentOS8 on the okd-svc host
   - Download qcow2 image
```
[root@sm-epyc-centos8 ~]# wget https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2 
``` 
2. Boot the okd-svc VM
```
[root@sm-epyc-centos8 ~]# /usr/bin/virt-install --connect qemu:///system --name okd-svc --memory 4096 --vcpus 4 --os-type=linux --disk /var/lib/libvirt/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2,device=disk,bus=virtio,format=qcow2 --network network:okd45,model=virtio --noautoconsole --import --os-variant rhel8.0 &> /dev/null
``` 
3. Move the files downloaded from the RedHat Cluster Manager site to the okd-svc node

```
[root@sm-epyc-centos8 ~]# scp ~/Downloads/openshift-install-linux.tar.gz ~/Downloads/openshift-client-linux.tar.gz root@{okd-svc_IP_address}:/root/
   ```

4. SSH to the okd-svc vm

```
[root@sm-epyc-centos8 ~]# ssh root@{okd-svc_IP_address}
[root@okd-svc ~]#
```

5. Extract Client tools and copy them to `/usr/local/bin`

```
[root@okd-svc ~]# mkdir /root/bin/
[root@okd-svc ~]# tar xvf openshift-client-linux.tar.gz -C /root/bin
```

6. Confirm Client Tools are working

```
[root@okd-svc ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T21:04:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18+", GitVersion:"v1.18.3", GitCommit:"9cdca7b", GitTreeState:"clean", BuildDate:"2020-10-02T23:12:03Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
```
```
[root@okd-svc ~]# oc version
Client Version: 4.5.0-202005291417-9933eb9
Server Version: 4.5.0-0.okd-2020-10-03-012432
Kubernetes Version: v1.18.3
```

7. Extract the OpenShift Installer

```
[root@okd-svc ~]# tar xvf openshift-install-linux.tar.gz -C /root/bin/
```

8. Update CentOS so we get the latest packages for each of the services we are about to install

```
[root@okd-svc ~]# dnf update
```

9. Install Git

```
[root@okd-svc ~]# dnf install git -y
```

10. Download [config files](git@github.com:vijayibm/okd4.5-setup.git) for each of the services

```
[root@okd-svc ~]# git clone git@github.com:vijayibm/okd4.5-setup.git
```

11. OPTIONAL: Create a file '~/.vimrc' and paste the following (this helps with editing in vim, particularly yaml files):

```
[root@okd-svc ~]# cat <<EOT >> ~/.vimrc
   syntax on
   set nu et ai sts=0 ts=2 sw=2 list hls
   EOT
```

Update the preferred editor

```
[root@okd-svc ~]# export OC_EDITOR="vim"
[root@okd-svc ~]# export KUBE_EDITOR="vim"
```

12. Set a Static IP for OKD network interface `nmtui-edit ens3` or edit `/etc/sysconfig/network-scripts/ifcfg-ens3`

   - **Address**: 192.168.100.1
   - **DNS Server**: 192.168.100.1
   - **Search domain**: okd45.smcloud.local

   > If changes arent applied automatically you can bounce the NIC with `nmcli connection down ens3` and `nmcli connection up ens3`

13. Setup firewalld

   Create **internal** and **external** zones

```
[root@okd-svc ~]# nmcli connection modify ens3 connection.zone internal
[root@okd-svc ~]# nmcli connection modify ens3 connection.zone external
```

   View zones:

```
[root@okd-svc ~]# firewall-cmd --get-active-zones
```

   Set masquerading (source-nat) on the both zones.

   So to give a quick example of source-nat - for packets leaving the external interface, which in this case is ens192 - after they have been routed they will have their source address altered to the interface address of ens192 so that return packets can find their way back to this interface where the reverse will happen.

```
[root@okd-svc ~]# firewall-cmd --zone=external --add-masquerade --permanent
[root@okd-svc ~]# firewall-cmd --zone=internal --add-masquerade --permanent
```

   Reload firewall config

```
[root@okd-svc ~]# firewall-cmd --reload
```

   Check the current settings of each zone

```
[root@okd-svc ~]# firewall-cmd --list-all --zone=internal
[root@okd-svc ~]# firewall-cmd --list-all --zone=external
```

   When masquerading is enabled so is ip forwarding which basically makes this host a router. Check:

```
[root@okd-svc ~]# cat /proc/sys/net/ipv4/ip_forward
```

14. Install and configure BIND DNS

   Install

```
[root@okd-svc ~]# dnf install bind bind-utils -y
```

   Apply configuration

```
[root@okd-svc ~]# cp ~/okd4.5-setup/dns/named.conf /etc/named.conf

[root@okd-svc ~]# cp -R ~/okd4.5-setup/dns/zones/* /var/named/
```
view config file: `/etc/named.conf`

```
[root@okd-svc ~]# cat /etc/named.conf
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
	listen-on port 53 { 127.0.0.1; any; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	secroots-file	"/var/named/data/named.secroots";
	recursing-file	"/var/named/data/named.recursing";
	allow-query     { localhost; any; };
        forwarders { 8.8.8.8; 8.8.4.4; };

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

	dnssec-enable yes;
	dnssec-validation yes;

	managed-keys-directory "/var/named/dynamic";

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
zone "smcloud.local" IN {
        type master;
        file "forword.zone";
        allow-update { none; };
};

zone "100.168.192.in-addr.arpa" IN {
        type master;
        file "reverse.zone";
        allow-update { none; };
};


include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

view zone files: `/var/named/forward.zone`
```
[root@okd-svc ~]# cat /var/named/forward.zone
$TTL 1D
@	IN SOA	 okd-svc.smcloud.local. root.smcloud.local. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
                 		IN NS           okd-svc.smcloud.local.
okd-svc 			IN A		192.168.100.1

$ORIGIN okd45.smcloud.local.
; Temp Bootstrap Node
okd-bootstrap			IN A		192.168.100.20

; Control Plane Nodes
okd-cp-1			IN A		192.168.100.21
okd-cp-2			IN A		192.168.100.22
okd-cp-3			IN A		192.168.100.23

; ETCD Cluster
etcd-0				IN A		192.168.100.21
etcd-1				IN A		192.168.100.22
etcd-2				IN A		192.168.100.23

; Worker Nodes
okd-w-1				IN A		192.168.100.24
okd-w-2				IN A		192.168.100.25

; OpenShift Internal SRV records (cluster name = okd45)
_etcd-server-ssl._tcp.okd45.smcloud.local.    86400     IN    SRV     0    10    2380    etcd-0
_etcd-server-ssl._tcp.okd45.smcloud.local.    86400     IN    SRV     0    10    2380    etcd-1
_etcd-server-ssl._tcp.okd45.smcloud.local.    86400     IN    SRV     0    10    2380    etcd-2

; OpenShift Internal - Load balancer
api       			IN A            192.168.100.10
api-int       			IN A            192.168.100.10

$ORIGIN apps.okd45.smcloud.local.
*				IN A		192.168.100.10

oauth-openshift     		IN A     	192.168.100.10
console-openshift-console     	IN A     	192.168.100.10
```
view zone files: `/var/named/reverse.zone`
```
[root@okd-svc ~]# cat /var/named/reverse.zone
$TTL 1D
@	IN SOA	 okd-svc.smcloud.local. root.smcloud.local. (

					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
                 		IN NS           okd-svc.smcloud.local.
1 				IN PTR 		okd-svc

; Temp Bootstrap Node
20				IN PTR		okd-bootstrap.okd45.smcloud.local.

; Control Plane Nodes
21 				IN PTR 		okd-cp-1.okd45.smcloud.local.
22 				IN PTR 		okd-cp-2.okd45.smcloud.local.
23 				IN PTR 		okd-cp-3.okd45.smcloud.local.

; ETCD Cluster
21 				IN PTR 		etcd-0.okd45.smcloud.local.
22 				IN PTR 		etcd-1.okd45.smcloud.local.
23 				IN PTR 		etcd-2.okd45.smcloud.local.

; Worker Nodes
24 				IN PTR 		okd-w-1.okd45.smcloud.local.
25 				IN PTR 		okd-w-2.okd45.smcloud.local.

; OpenShift Internal - Load balancer
10				IN PTR		api.okd45.smcloud.local
10				IN PTR		api-api.okd45.smcloud.local
```



   Configure the firewall for DNS

```
[root@okd-svc ~]# firewall-cmd --add-port=53/udp --zone=internal --permanent
[root@okd-svc ~]# firewall-cmd --reload
```

   Enable and start the service

```
[root@okd-svc ~]# systemctl enable named
[root@okd-svc ~]# systemctl start named
[root@okd-svc ~]# systemctl status named
```

   Confirm dig now sees the correct DNS results by using the DNS Server running locally

```
[root@okd-svc ~]# dig okd45.smcloud.local
```
The following should return the answer okd-bootstrap.okd45.smcloud.local from the local server

```
[root@okd-svc ~]# dig -x 192.168.100.20
; <<>> DiG 9.11.13-RedHat-9.11.13-6.el8_2.1 <<>> -x 192.168.100.20
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21483
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 44ff2d5af56a9a091c08932a5f8be60f56018037b5fecf82 (good)
;; QUESTION SECTION:
;20.100.168.192.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
20.100.168.192.in-addr.arpa. 86400 IN	PTR	okd-bootstrap.okd45.smcloud.local.

;; AUTHORITY SECTION:
100.168.192.in-addr.arpa. 86400	IN	NS	okd-svc.smcloud.local.

;; ADDITIONAL SECTION:
okd-svc.smcloud.local. 86400 IN	A	192.168.100.5

;; Query time: 0 msec
;; SERVER: 10.160.10.1#53(10.160.10.1)
;; WHEN: Sun Oct 18 12:21:59 IST 2020
;; MSG SIZE  rcvd: 177
```

15. Install & configure DHCP

   Install the DHCP Server

```
[root@okd-svc ~]# dnf install dhcp-server -y
```

   Edit dhcpd.conf from the cloned git repo to have the correct mac address for each host and copy the conf file to the correct location for the DHCP service to use

```
[root@okd-svc ~]# cp ~/okd4.5-setup/dhcpd.conf /etc/dhcp/dhcpd.conf
```
view `/etc/dhcp/dhcpd.conf` file:

```
[root@okd-svc ~]# cat /etc/dhcp/dhcpd.conf
authoritative;
ddns-update-style interim;
allow booting;
allow bootp;
allow unknown-clients;
ignore client-updates;
default-lease-time 14400;
max-lease-time 14400;

subnet 192.168.100.0 netmask 255.255.255.0 {
 option routers                  192.168.100.1; # lan
 option subnet-mask              255.255.255.0;
 option domain-name              "okd45.smcloud.local";
 option domain-name-servers       192.168.100.1;
 range 192.168.100.11 192.168.100.50;
}

host okd-bootstrap {
 hardware ethernet 52:54:00:3e:30:af;
 fixed-address 192.168.100.20;
 option host-name "okd-bootstrap";
}

host okd-cp-1 {
 hardware ethernet 52:54:00:df:d3:b0;
 fixed-address 192.168.100.21;
 option host-name "okd-cp-1";
}

host okd-cp-2 {
 hardware ethernet 52:54:00:39:1f:46;
 fixed-address 192.168.100.22;
 option host-name "okd-cp-2";
}

host okd-cp-3 {
 hardware ethernet 52:54:00:08:24:8e;
 fixed-address 192.168.100.23;
 option host-name "okd-cp-3";
}

host okd-w-1 {
 hardware ethernet 52:54:00:8d:9d:77;
 fixed-address 192.168.100.24;
 option host-name "okd-w-1";
}

host okd-w-2 {
 hardware ethernet 52:54:00:6f:39:cd;
 fixed-address 192.168.100.25;
 option host-name "okd-w-2";
}

host okd-haproxy {
 hardware ethernet 52:54:00:f6:10:dc;
 fixed-address 192.168.100.10;
 option host-name "okd-haproxy";
}
```

   Configure the Firewall

```
[root@okd-svc ~]# firewall-cmd --add-service=dhcp --zone=internal --permanent
[root@okd-svc ~]# firewall-cmd --reload
```

   Enable and start the service

```
[root@okd-svc ~]# systemctl enable dhcpd
[root@okd-svc ~]# systemctl start dhcpd
[root@okd-svc ~]# systemctl status dhcpd
```

16. Install & configure Apache Web Server

   Install Apache

```
[root@okd-svc ~]# dnf install httpd -y
```
```
[root@okd-svc ~]# firewall-cmd --add-port=80/tcp --zone=internal --permanent
[root@okd-svc ~]# firewall-cmd --reload
```

   Enable and start the service

```
[root@okd-svc ~]# systemctl enable httpd
[root@okd-svc ~]# systemctl start httpd
[root@okd-svc ~]# systemctl status httpd
```

17. Install & configure HAProxy

   Install HAProxy

```
[root@okd-haproxy ~]# dnf install haproxy -y
```

   Copy HAProxy config

```
[root@okd-haproxy ~]# cp ~/okd4.5/haproxy.cfg /etc/haproxy/haproxy.cfg
```
view `/etc/haproxy/haproxy.cfg`

```
[root@okd-haproxy ~]# cat /etc/haproxy/haproxy.cfg
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    log                     global
    mode                    http
    option                  httplog
    option                  dontlognull
    option http-server-close
    option redispatch
    option forwardfor       except 127.0.0.0/8
    retries                 3
    maxconn                 20000
    timeout http-request    10000ms
    timeout http-keep-alive 10000ms
    timeout check           10000ms
    timeout connect         40000ms
    timeout client          300000ms
    timeout server          300000ms
    timeout queue           50000ms

# Enable HAProxy stats
listen stats
    bind :9000
    stats uri /stats
    stats refresh 10000ms

# Kube API Server
frontend k8s_api_frontend
    bind :6443
    default_backend k8s_api_backend
    mode tcp

backend k8s_api_backend
    mode tcp
    balance source
    server      okd-bootstrap 192.168.100.20:6443 check
    server      okd-cp-1 192.168.100.21:6443 check
    server      okd-cp-2 192.168.100.22:6443 check
    server      okd-cp-3 192.168.100.23:6443 check

# OCP Machine Config Server
frontend okd_machine_config_server_frontend
    mode tcp
    bind :22623
    default_backend okd_machine_config_server_backend

backend okd_machine_config_server_backend
    mode tcp
    balance source
    server      okd-bootstrap 192.168.100.20:22623 check
    server      okd-cp-1 192.168.100.21:22623 check
    server      okd-cp-2 192.168.100.22:22623 check
    server      okd-cp-3 192.168.100.23:22623 check

# OCP Ingress - layer 4 tcp mode for each. Ingress Controller will handle layer 7.
frontend okd_http_ingress_frontend
    bind :80
    default_backend okd_http_ingress_backend
    mode tcp

backend okd_http_ingress_backend
    balance source
    mode tcp
    server      okd-w-1 192.168.100.24:80 check
    server      okd-w-2 192.168.100.25:80 check

frontend okd_https_ingress_frontend
    bind *:443
    default_backend okd_https_ingress_backend
    mode tcp

backend okd_https_ingress_backend
    mode tcp
    balance source
    server      okd-w-1 192.168.100.24:443 check
    server      okd-w-2 192.168.100.25:443 check
```

   Configure the Firewall

   > Note: Opening port 9000 in the external zone allows access to HAProxy stats that are useful for monitoring and troubleshooting. The UI can be accessed at: `http://{okd-haproxy_IP_Address}:9000/stats`

```
[root@okd-haproxy ~]# firewall-cmd --add-port=6443/tcp --zone=internal --permanent # kube-api-server on control plane nodes
[root@okd-haproxy ~]# firewall-cmd --add-port=6443/tcp --zone=external --permanent # kube-api-server on control plane nodes
[root@okd-haproxy ~]# firewall-cmd --add-port=22623/tcp --zone=internal --permanent # machine-config server
[root@okd-haproxy ~]# firewall-cmd --add-service=http --zone=internal --permanent # web services hosted on worker nodes
[root@okd-haproxy ~]# firewall-cmd --add-service=http --zone=external --permanent # web services hosted on worker nodes
[root@okd-haproxy ~]# firewall-cmd --add-service=https --zone=internal --permanent # web services hosted on worker nodes
[root@okd-haproxy ~]# firewall-cmd --add-service=https --zone=external --permanent # web services hosted on worker nodes
[root@okd-haproxy ~]# firewall-cmd --add-port=9000/tcp --zone=external --permanent # HAProxy Stats
[root@okd-haproxy ~]# firewall-cmd --reload
```

   Enable and start the service

```
[root@okd-haproxy ~]# setsebool -P haproxy_connect_any 1 # SELinux name_bind access
[root@okd-haproxy ~]# systemctl enable haproxy
[root@okd-haproxy ~]# systemctl start haproxy
[root@okd-haproxy ~]# systemctl status haproxy
```

18. Install and configure NFS for the OpenShift Registry. It is a requirement to provide storage for the Registry, emptyDir can be specified if necessary.

   Install NFS Server

```
[root@okd-svc ~]# dnf install nfs-utils -y
```

   Create the Share

   Check available disk space and its location `df -h`

```
[root@okd-svc ~]# mkdir -p /okd45/registry
[root@okd-svc ~]# chown -R nobody:nobody /okd45/registry
[root@okd-svc ~]# chmod -R 777 /okd45/registry
   ```

   Export the Share

   ```
[root@okd-svc ~]# echo "/okd45/registry  192.168.100.0/24(rw,sync,root_squash,no_subtree_check,no_wdelay)" > /etc/exports
   exportfs -rv
   ```

   Set Firewall rules:

```
[root@okd-svc ~]# firewall-cmd --zone=internal --add-service mountd --permanent
[root@okd-svc ~]# firewall-cmd --zone=internal --add-service rpc-bind --permanent
[root@okd-svc ~]# firewall-cmd --zone=internal --add-service nfs --permanent
[root@okd-svc ~]# firewall-cmd --reload
```

   Enable and start the NFS related services

```
[root@okd-svc ~]# systemctl enable nfs-server rpcbind
[root@okd-svc ~]# systemctl start nfs-server rpcbind nfs-mountd
```

## Generate and host install files

20. Generate an SSH key pair keeping all default options

```
[root@okd-svc ~]# ssh-keygen
```

21. Create an install directory

```
[root@okd-svc ~]# mkdir ~/okd-install
```

22. Copy the install-config.yaml included in the clones repository to the install directory

```
[root@okd-svc ~]# cp ~/okd4.5-setup/install-config.yaml ~/okd-install
```

23. Update the install-config.yaml with your own ssh key.

   - Line 24 should contain the contents of your '~/.ssh/id_rsa.pub'

```
[root@okd-svc ~]# cat ~/okd-install/install-config.yaml
```

24. Generate Kubernetes manifest files

```
[root@okd-svc ~]#openshift-install create manifests --dir ~/okd-install
```

   > A warning is shown about making the control plane nodes schedulable. It is up to you if you want to run workloads on the Control Plane nodes. If you dont want to you can disable this with:
```
[root@okd-svc ~]# sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' ~/okd-install/manifests/cluster-scheduler-02-config.yml
```
   > Make any other custom changes you like to the core Kubernetes manifest files.

   Generate the Ignition config and Kubernetes auth files

```
[root@okd-svc ~]#openshift-install create ignition-configs --dir ~/okd-install/
```

25. Create a hosting directory to serve the configuration files for the OpenShift booting process

```
[root@okd-svc ~]# mkdir /var/www/html/okd4
```

26. Copy all generated install files to the new web server directory

```
[root@okd-svc ~]# cp -R ~/okd-install/* /var/www/html/okd4
```

27. Change ownership and permissions of the web server directory

```
[root@okd-svc ~]# chcon -R -t httpd_sys_content_t /var/www/html/okd4/
[root@okd-svc ~]# chown -R apache: /var/www/html/okd4/
[root@okd-svc ~]# chmod 644 /var/www/html/okd4/
```

28. Confirm you can see all files added to the `/var/www/html/okd4/` dir through Apache

```
[root@okd-svc ~]# curl -I http://localhost/okd4/
HTTP/1.1 200 OK
Date: Sun, 18 Oct 2020 07:18:50 GMT
Server: Apache/2.4.37 (centos)
Last-Modified: Thu, 15 Oct 2020 08:13:34 GMT
ETag: "48d22-5b1b1396be415"
Accept-Ranges: bytes
Content-Length: 298274
Content-Type: application/vnd.coreos.ignition+json
```

## Deploy OpenShift

1. Create qcow2 image for okd-bootstrap host and okd-cp-\#.
```
[root@sm-epyc-centos8 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/okd-bootstrap.qcow2 120G

[root@sm-epyc-centos8 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/okd-cp-1.qcow2 120G

[root@sm-epyc-centos8 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/okd-cp-2.qcow2 120G

[root@sm-epyc-centos8 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/okd-cp-3.qcow2 120G

[root@sm-epyc-centos8 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/okd-w-1.qcow2 120G

[root@sm-epyc-centos8 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/okd-w-2.qcow2 120G

```
2. Boot the okd-bootstrap , okd-cp-\# and okd-w-\# hosts with `FCOS live` ISO image and enter the following configuration:
```
 # Bootstrap Node - okd-bootstrap
 coreos-installer install --insecure-ignition --ignition-url=https://192.168.100.5/bootstrap.ign /dev/vda
```
```
 # Each of the Control Plane Nodes - okd-cp-\#
 coreos-installer install --insecure-ignition --ignition-url=https://192.168.100.5/master.ign /dev/vda
```
```
 # Each of the Worker Nodes - okd-w-\#
 coreos-installer install --insecure-ignition --ignition-url=https://192.168.100.5/worker.ign /dev/vda
```
Once Installtion done reboot all the VMs.

##### NOTE: After FCOS installs, the system reboots. During the system reboot, it applies the Ignition config file that you specified.

## Monitor the Bootstrap Process

3. You can monitor the bootstrap process from the okd-svc host at different log levels (debug, error, info)

```
[root@okd-svc ~]# openshift-install --dir ~/okd-install wait-for bootstrap-complete --log-level=debug
```

4. Once bootstrapping is complete the okd-boostrap node [can be removed](#remove-the-bootstrap-node)

## Remove the Bootstrap Node

5. Remove all references to the `okd-bootstrap` host from the `/etc/haproxy/haproxy.cfg` file

```
   # Two entries
[root@okd-svc ~]# vim /etc/haproxy/haproxy.cfg

   # Restart HAProxy - If you are still watching HAProxy stats console you will see that the okd-boostrap host has been removed from the backends.

[root@okd-svc ~]# systemctl reload haproxy
   ```

6. The okd-bootstrap host can now be safely shutdown and deleted from the KVM Host, the host is no longer required

## Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

1. Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event

```
[root@okd-svc ~]# openshift-install --dir ~/okd-install wait-for install-complete
```

2. Continue to join the worker nodes to the cluster in a new tab whilst waiting for the above command to complete

## Join Worker Nodes

3. Setup 'oc' and 'kubectl' clients on the okd-svc machine

```
[root@okd-svc ~]# export KUBECONFIG=~/okd-install/auth/kubeconfig

   # Test auth by viewing cluster nodes

[root@okd-svc ~]# oc get nodes
NAME       STATUS                     ROLES    AGE     VERSION
okd-cp-1   Ready                      master   2d23h   v1.18.3
okd-cp-2   Ready                      master   2d23h   v1.18.3
okd-cp-3   Ready                      master   2d23h   v1.18.3
```

1. View and approve pending CSRs

   > Note: Once you approve the first set of CSRs additional 'kubelet-serving' CSRs will be created. These must be approved too.
   > If you do not see pending requests wait until you do.

```
   # View CSRs
[root@okd-svc ~]# oc get csr

   # Approve all pending CSRs
[root@okd-svc ~]# oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

   # Wait for kubelet-serving CSRs and approve them too with the same command
[root@okd-svc ~]# oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   ```

1. Watch and wait for the Worker Nodes to join the cluster and enter a 'Ready' status

   > This can take 5-10 minutes

```
[root@okd-svc ~]# oc get nodes
NAME       STATUS                     ROLES    AGE     VERSION
okd-cp-1   Ready                      master   2d23h   v1.18.3
okd-cp-2   Ready                      master   2d23h   v1.18.3
okd-cp-3   Ready                      master   2d23h   v1.18.3
okd-w-1    Ready		      worker   2d22h   v1.18.3
okd-w-2    Ready                      worker   2d21h   v1.18.3
```

## Configure storage for the Image Registry

> A Bare Metal cluster does not by default provide storage so the Image Registry Operator bootstraps itself as 'Removed' so the installer can complete. As the installation has now completed storage can be added for the Registry and the operator updated to a 'Managed' state.

1. Create the 'image-registry-storage' PVC by updating the Image Registry operator config by updating the management state to 'Managed' and adding 'pvc' and 'claim' keys in the storage key:

```
[root@okd-svc ~]# oc edit configs.imageregistry.operator.openshift.io
```

   ```yaml
   managementState: Managed
   ```

   ```yaml
   storage:
     pvc:
       claim: # leave the claim blank
   ```

2. Confirm the 'image-registry-storage' pvc has been created and is currently in a 'Pending' state

```
[root@okd-svc ~]# oc get pvc -n openshift-image-registry
```

3. Create the persistent volume for the 'image-registry-storage' pvc to bind to

```yaml
[root@okd-svc ~]# cat ~/okd4.5-setup/manifest/registry-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Gi
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /okd45/registry
    server: 192.168.100.5
```
```
[root@okd-svc ~]# oc create -f ~/okd4.5-setup/manifest/registry-pv.yaml
```

4. After a short wait the 'image-registry-storage' pvc should now be bound

```
[root@okd-svc ~]# oc get pvc -n openshift-image-registry
```

## Create the first Admin user

5. Apply the `oauth-htpasswd.yaml` file to the cluster

   > This will create a user 'admin' with the password 'password'. To set a different username and password substitue the htpasswd key in the '~/okd4-metal-install/manifest/oauth-htpasswd.yaml' file with the output of `htpasswd -n -B -b <username> <password>`

```
[root@okd-svc ~]# oc apply -f ~/okd4.5-setup/manifest/oauth-htpasswd.yaml
```

6. Assign the new user (admin) admin permissions

```
[root@okd-svc ~]# oc adm policy add-cluster-role-to-user cluster-admin admin
```

## Access the OpenShift Console

7. Wait for the 'console' Cluster Operator to become available

```
[root@okd-svc ~]# oc get co
```

8. Append the following to your local workstations `/etc/hosts` file:

   > From your local workstation
   > If you do not want to add an entry for each new service made available on OpenShift you can configure the okd-svc DNS server to serve externally and create a wildcard entry for \*.apps.okd45.smcloud.local

```
   # Open the hosts file
sudo vi /etc/hosts

   # Append the following entries:
   192.168.0.96 okd-svc api.okd45.smcloud.local console-openshift-console.apps.okd45.smcloud.local oauth-openshift.apps.okd45.smcloud.local downloads-openshift-console.apps.okd45.smcloud.local alertmanager-main-openshift-monitoring.apps.okd45.smcloud.local grafana-openshift-monitoring.apps.okd45.smcloud.local prometheus-k8s-openshift-monitoring.apps.okd45.smcloud.local thanos-querier-openshift-monitoring.apps.okd45.smcloud.local
   ```

9. Navigate to the [OpenShift Console URL](https://console-openshift-console.apps.okd45.smcloud.local) and log in as the 'admin' user

   > You will get self signed certificate warnings that you can ignore
   > If you need to login as kubeadmin and need to the password again you can retrieve it with: `cat ~/okd-install/auth/kubeadmin-password`

## Troubleshooting

10. You can collect logs from all cluster hosts by running the following command from the 'okd-svc' host:

```
[root@okd-svc ~]# openshift-install gather bootstrap --dir okd-install --bootstrap=192.168.100.20 --master=192.168.100.21 --master=192.168.100.22 --master=192.168.100.23
```

11. Modify the role of the Control Plane Nodes

   If you would like to schedule workloads on the Control Plane nodes apply the 'worker' role by changing the value of 'mastersSchedulable' to true.

   If you do not want to schedule workloads on the Control Plane nodes remove the 'worker' role by changing the value of 'mastersSchedulable' to false.

   > Remember depending on where you host your workloads you will have to update HAProxy to include or exclude the control plane nodes from the ingress backends.

```
[root@okd-svc ~]# oc edit schedulers.config.openshift.io cluster
```
