# RKE2-3nodeCluster
RKE2 3 node Kubernetes 3x control-plane,etcd,master with rancher helm install install with NGINX load balancer with troubleshooting tips
I was unable to find a guide on how to do this with 3 master,control-plane,etcd. I was only able to find guides with single master,control-plane, etcd with agent nodes.
This guide was created using Rocky linux 9.3, but should work with any linux distro as a base OS without major changes
I hope this guide helps

# Arctiecture

``
Nginx Load balancer = 192.168.1.100
``
``
RKE2-server1 = 192.168.1.101
``
``
RKE2-server2 = 192.168.1.102
``
``
RKE2-server3 = 192.168.1.103
``

## Rational 
this architecture is so that if one of the control planes does go down, we can still query the api and have High availability.
If you run only 1 control-plane and that fails, you will no longer have access to the kubectl API

#Nginx load balancer – 192.168.1.100
On the server run the following

``
dnf update -y
dnf intally -y nginx nginx-mod-stream

``
In the directory above, I have made a sample config file, if you copy that and paste it into /etc/nginx/nginx.conf
If you are comfortable using your own load balancer feel free to do it, but it is important especially for the setting up the cluster, that ports 6443 and 9345 are able to be load balanced for connecting the cluster otherwise you will have issues and it will not connect.
I have added in http and https there too for the next steps so you can see rancher or any other app you wish to add the local cluster using those ports.

## ALL RKE2 servers
I recommend turning off the firewall on the initial setup and then refining it afterwards as Kubernetes has a lot of ports and finding out which ports you need will be a lot of work.
You can find documentation on port requirements
Requirements | RKE2
Port Requirements | Rancher
You do not need to turn off selinux, the install works well with it. 
dnf update -y
dnf install -y  nfs-utils cryptsetup iscsi-initiator-utils tar

## RKE2 server 1
For this server run the following

```
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.26 INSTALL_RKE2_TYPE=server sh –
```

After it installs, create /etc/rancher/rke2/config.yaml and add all ip addresses of each server.


![image](https://github.com/R3m1xed/RKE2-3nodeCluster/assets/80881749/f1133285-07f5-4427-95b9-e93fa460e57b)

Now run:

```
systemctl enable rke2-server –now
```

This may take a minute or 2 to complete as this starts the process of configuring the new node
After this has completed. Run: 

```
#needed so you can use the kubectl api
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl 
kubectl get nodes
```

If it is not working, make sure to run the export KUBECONFIG line again and try again.
The finish this off we need to grab the server token. This is located in /var/lib/rancher/rke2/server/token
Copy that whole token and save it to a notepad for the other 2 servers as you will need it to join cluster
#RKE2 servers 1 and 2
Run the following

```
curl -sfL https://get.rke2.io | sh –
```

After this create /etc/rancher/rke2/config.yaml and add the following in there

![image](https://github.com/R3m1xed/RKE2-3nodeCluster/assets/80881749/9917852c-102d-481a-a22b-1c2b838082d9)

Now what we need to do is edit the systemd service that runs this as will not check config file that we created. Modify /usr/lib/systemd/system/rke2-server.service. In particular, the only line we editing in the file is the “ExecStart”

```

[Unit]
Description=Rancher Kubernetes Engine v2 (server)
Documentation=https://github.com/rancher/rke2#readme
Wants=network-online.target
After=network-online.target
Conflicts=rke2-agent.service

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-/usr/lib/systemd/system/%N.env
KillMode=process
Delegate=yes
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=/bin/sh -xc '! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service'
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/rke2 server -c /etc/rancher/rke2/config.yaml
ExecStopPost=-/bin/sh -c "systemd-cgls /system.slice/%n | grep -Eo '[0-9]+ (containerd|kubelet)' | awk '{print $1}' | xargs -r kill"

```

Now run systemctl start rke2-server
Note: if this takes more than 5 minutes and forever hangs, cancel it, reboot that machine and run rke2 server -c /etc/rancher/rke2/config.yaml and find where it is getting stuck
If it has gone through with no errors run the following and check to make sure your new node is appearing with the same roles as the first node

``
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) /usr/local/bin/kubectl 
kubectl get nodes
``

Repeat this on the other server

*** installing helm, rancher and certmanager
Helm should be installed on machines, we can run the following on all servers to install helm

```
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

On 1 of rke2 servers run the following

```
*** adds repos
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io
*** install cert manager
helm upgrade -i cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set installCRDs=true
*** install rancher – not change the hostname in the command to a name you would like to use
helm upgrade -i rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=rancher.linuxezy.net --set bootstrapPassword=RKE2isawesome --set replicas=1
```

Now make sure to create a dns record or modify your hosts file to of the rancher hostname to point to the load balancer (192.168.1.100)

#Troubleshooting tips
1.	Rke service not starting or failing:
After you have installed rke2, if starting service has not gone well. You will need to reboot your server first as I’ve found it holds up port 9345
Run rke2 server -c /location/of/config.yaml. This will help greatly and telling you where it is getting stuck.

2.	Your config is not being read when running systemctl start rke2-server
Make sure to edit the system service, /usr/lib/systemd/system/rke2-server.service.
Modify it so it runs – rke2 server -c /location/of/config.yaml

3.	Getting a CA identity issue
Likely you have either forgotten the tls-san option in the config file, or your load balancer is not set up correctly.
run curl –insecure https://<lb-ip>/ca-location to verify, the correct url will appear when running rke2 server -c /location/of/config



