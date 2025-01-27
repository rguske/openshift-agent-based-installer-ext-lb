
# Installing OpenShift using the Agent-Based Installer with an external Load Balancer - Platform none

Install OpenShift Container Platform on any platform.

[Docs - Installing on any platform](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/installing_on_any_platform/installing-platform-agnostic#installation-load-balancing-user-infra_installing-platform-agnostic)

> In the `install-config.yaml`, specify the platform on which to perform the installation. The following platforms are supported:
>
> * `baremetal`
> * `vsphere`
> * `none`

> Important
> For platform none:
>
> The none option requires the provision of DNS name resolution and load balancing infrastructure in your cluster. See Requirements for a cluster using the platform "none" option in the "Additional resources" section for more information.
Review the information in the guidelines for deploying OpenShift Container Platform on non-tested platforms before you attempt to install an OpenShift Container Platform cluster in virtualized or cloud environments.

## Preperations

Setup a Bastion Host using e.g RHEL9.

On the bastion host, download the necessary cli's:

`curl -LO <url>`

- [openshift-install-rhel9](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.17.6/openshift-install-rhel9-amd64.tar.gz)
- [openshift-client-linux-amd64](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.17.6/openshift-client-linux-amd64-rhel9-4.17.6.tar.gz)

Unpack the `.gz`files and copy them into your path:

If /usr/local/bin isn't included in the $PATH, run
`export PATH=/usr/local/bin:$PATH`

```shell
cp openshift-install /usr/local/bin/
cp oc /usr/local/bin/
cp kubectl /usr/local/bin/
```

## Validation via RHEL Live iso

In order to get the nic interface names of your servers, it could be helpful to quickly run a live-iso RHEL.

`curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.17/latest/rhcos-4.17.2-x86_64-live.x86_64.iso`

Run:

`ip link show`

or `dmesg | grep -i eth` or `ls /sys/class/net`

## Bastion Host Preperation

`cat ~/.ssh/id_ed25519.pub | ssh rguske@rguske-bastion.rguske.coe.muc.redhat.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"`

`sudo subscription-manager register --username  --password `

## Setup an External Load Balancer

Before you install OpenShift Container Platform, you must provision the API and application Ingress load balancing infrastructure. In production scenarios, you can deploy the API and application Ingress load balancers separately so that you can scale the load balancer infrastructure for each in isolation.

For experimental, educational or PoC purposes, the following HAProxy container can come into handy: [OpenShift 4 load balancer container image](https://github.com/RedHat-EMEA-SSA-Team/openshift-4-loadbalancer).

This container can run as a service on your RHEL Bastion host. Install the [Container-Tools](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/building_running_and_managing_containers/index#proc_getting-container-tools_assembly_starting-with-containers) first, in order to have Podman as the container runtime available.

`dnf install container-tools`

Adjust the following `openshift-4-loadbalancer.service` file, to have the HAProxy container serving properly. Replace the IP addresses of your OCP cluster nodes, the IP for the cluster API as well as the Ingress IP accordingly.

For $API as well $INGRESS, **use the IP address of your Bastion host!**

```code
cat > /etc/systemd/system/openshift-4-loadbalancer.service <<EOF
[Unit]
Description=OpenShift 4 LoadBalancer CLUSTER
After=network.target

[Service]
Type=simple
TimeoutStartSec=5m

ExecStartPre=-/usr/bin/podman rm "openshift-4-loadbalancer"
ExecStartPre=/usr/bin/podman pull quay.io/redhat-emea-ssa-team/openshift-4-loadbalancer
ExecStart=/usr/bin/podman run --name openshift-4-loadbalancer --net host \
  -e API=cp1=10.32.96.122:6443,cp2=10.32.96.123:6443,cp3=10.32.96.124:6443 \
  -e API_LISTEN=127.0.0.1:6443,10.32.96.139:6443 \
  -e INGRESS_HTTP=cp1=10.32.96.122:80,cp2=10.32.96.123:80,cp3=10.32.96.124:80,n1=10.32.96.125:80,n2=10.32.96.126:80 \
  -e INGRESS_HTTP_LISTEN=127.0.0.1:80,10.32.96.139:80 \
  -e INGRESS_HTTPS=cp1=10.32.96.122:443,cp2=10.32.96.123:443,cp3=10.32.96.124:443,n1=10.32.96.125:443,n2=10.32.96.126:443 \
  -e INGRESS_HTTPS_LISTEN=127.0.0.1:443,10.32.96.139:443 \
  -e MACHINE_CONFIG_SERVER=cp1=10.32.96.122:22623,cp2=10.32.96.123:22623,cp3=10.32.96.124:22623 \
  -e MACHINE_CONFIG_SERVER_LISTEN=127.0.0.1:22623,10.32.96.139:22623 \
  -e STATS_LISTEN=127.0.0.1:1984 \
  -e STATS_ADMIN_PASSWORD=aengeo4oodoidaiP \
  -e HAPROXY_CLIENT_TIMEOUT=1m \
  -e HAPROXY_SERVER_TIMEOUT=1m \
  quay.io/redhat-emea-ssa-team/openshift-4-loadbalancer

ExecReload=-/usr/bin/podman stop "openshift-4-loadbalancer"
ExecReload=-/usr/bin/podman rm "openshift-4-loadbalancer"
ExecStop=-/usr/bin/podman stop "openshift-4-loadbalancer"
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

Start the service.

```shell
systemctl daemon-reload
systemctl enable --now openshift-4-loadbalancer.service
systemctl status openshift-4-loadbalancer.service
```

By starting the service, the `openshift-4-loadbalancer` container will be started.

```shell
podman ps
CONTAINER ID  IMAGE                                                         COMMAND               CREATED     STATUS      PORTS                                 NAMES
e43af3dbb66a  quay.io/redhat-emea-ssa-team/openshift-4-loadbalancer:latest  haproxy -f /hapro...  2 days ago  Up 2 days   80/tcp, 443/tcp, 6443/tcp, 22623/tcp  openshift-4-loadbalancer
```

Configure your firewall accordingly:

```shell
firewall-cmd --zone=public --permanent --add-port=80/tcp
firewall-cmd --zone=public --permanent --add-port=443/tcp
firewall-cmd --zone=public --permanent --add-port=6443/tcp
firewall-cmd --zone=public --permanent --add-port=22623/tcp
firewall-cmd --zone=public --permanent --add-port=1984/tcp
firewall-cmd --reload
```

## Cluster Preperations

* created 3 Control-Plane nodes
* created 2 Worker-Nodes

**IMPORTANT** configure the Advanced Parameters `disk.EnableUUID = True` when VMs got created manually on vSphere.

It'll be a HA OCP cluster. Three types of clusters are supported:

* Single-Node cluster (SNO)
* Compact cluster (three nodes - master and worker in one)
* HA cluster (three cp nodes and three worker nodes)

Collecting the necessary nic information:

| name  | nic | mac | ipv4 | comment |
|---|---|---|---|---|
| ocp1-cp1  | ens33 | 00:50:56:88:cf:65 | 10.32.96.122  | Control Plane Node 1  |
| ocp1-cp2  | ens33 |  00:50:56:88:d4:49 | 10.32.96.123  | Control Plane Node 2  |
| ocp1-cp3  | ens33 | 00:50:56:88:18:fe | 10.32.96.124  | Control Plane Node 3  |
| ocp1-n1  | ens33 | 00:50:56:88:08:96 | 10.32.96.125  | Worker Node 1  |
| ocp1-n2  | ens33 | 00:50:56:88:a7:eb | 10.32.96.126  | Worker Node 2  |

BaseDomain: rguske.coe.muc.redhat.com

## Configurations

`agent-config.yaml`

```yaml
cat > agent-config.yaml << EOF
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: ocp1
rendezvousIP: 10.32.96.122
hosts:
  - hostname: ocp1-cp1.rguske.coe.muc.redhat.com
    role: master
    interfaces:
      - name: ens33
        macAddress: 00:50:56:88:cf:65
    networkConfig:
      interfaces:
        - name: ens33
          type: ethernet
          state: up
          mac-address: 00:50:56:88:cf:65
          ipv4:
            enabled: true
            address:
              - ip: 10.32.96.122
                prefix-length: 20
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.32.96.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.32.111.254
            next-hop-interface: ens33
            table-id: 254
  - hostname: ocp1-cp2.rguske.coe.muc.redhat.com
    role: master
    interfaces:
      - name: ens33
        macAddress: 00:50:56:88:d4:49
    networkConfig:
      interfaces:
        - name: ens33
          type: ethernet
          state: up
          mac-address: 00:50:56:88:d4:49
          ipv4:
            enabled: true
            address:
              - ip: 10.32.96.123
                prefix-length: 20
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.32.96.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.32.111.254
            next-hop-interface: ens33
            table-id: 254
  - hostname: ocp1-cp3.rguske.coe.muc.redhat.com
    role: master
    interfaces:
      - name: ens33
        macAddress: 00:50:56:88:18:fe
    networkConfig:
      interfaces:
        - name: ens33
          type: ethernet
          state: up
          mac-address: 00:50:56:88:18:fe
          ipv4:
            enabled: true
            address:
              - ip: 10.32.96.124
                prefix-length: 20
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.32.96.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.32.111.254
            next-hop-interface: ens33
            table-id: 254
  - hostname: ocp1-n1.rguske.coe.muc.redhat.com
    role: worker
    interfaces:
      - name: ens33
        macAddress: 00:50:56:88:08:96
    networkConfig:
      interfaces:
        - name: ens33
          type: ethernet
          state: up
          mac-address: 00:50:56:88:08:96
          ipv4:
            enabled: true
            address:
              - ip: 10.32.96.125
                prefix-length: 20
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.32.96.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.32.111.254
            next-hop-interface: ens33
            table-id: 254
  - hostname: ocp1-n2.rguske.coe.muc.redhat.com
    role: worker
    interfaces:
      - name: ens33
        macAddress: 00:50:56:88:a7:eb
    networkConfig:
      interfaces:
        - name: ens33
          type: ethernet
          state: up
          mac-address: 00:50:56:88:a7:eb
          ipv4:
            enabled: true
            address:
              - ip: 10.32.96.126
                prefix-length: 20
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.32.96.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.32.111.254
            next-hop-interface: ens33
            table-id: 254
EOF
```

`install-config.yaml`

```yaml
cat > install-config.yaml << EOF
apiVersion: v1
baseDomain: rguske.coe.muc.redhat.com
compute:
- name: worker
  replicas: 2
controlPlane:
  name: master
  replicas: 3
metadata:
  name: ocp1
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 10.32.96.0/20
  serviceNetwork:
    - 172.30.0.0/16
  networkType: OVNKubernetes
platform:
  none: {}
fips: false
pullSecret: '{"auths":{"cloud.openshift.com":...'
sshKey: 'ssh-rsa AAAAB3Nz...'
EOF
```

## Create Agent iso

`mkdir conf`

Create the install-config.yaml and agent-install.yaml file.

Run `openshift-install agent create image --dir conf/`

Mount the `agent.x86_64.iso` on the machines (BM or VM).

Boot the machines and wait until the installation is done.

Validate the installer progress using `openshift-install wait-for install-complete --dir conf/`

## Run a `httpd` webserver on the bastion to share the iso

Depending on your environment, providing the created iso can be cumbersome.

One quick and easy way could be by making it downloadable via a webserver.

Install `httpd` on the bastion host.

```bash
dnf install httpd
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Validate the service is running:

```bash
sudo ss -tuln | grep :80
curl -I http://localhost
sudo tail -f /var/log/httpd/error_log
```

Copy the created iso into `/var/www/html/`.

Download the iso by using e.g. `curl -LO http://<bastion-name/ip>/agent.x86_64.iso` or `wget`.

**IMPORTANT!**: The `httpd` service is listening on port 80. So the `openshift-4-loadbalancer` service. Stop the load balancer service for the time you exchange the created `agent.iso`.

## Validate HAProxy is serving

You can verify that the HAProxy container is serving your services by executing `podman exec -ti openshift-4-loadbalancer /watch-stats.sh` on your bastion host.

```code
stats                  FRONTEND  OPEN
ingress-http           FRONTEND  OPEN
ingress-http           cp1       DOWN    Connection refused
ingress-http           cp2       DOWN    Connection refused
ingress-http           cp3       DOWN    Connection refused
ingress-http           n1        UP
ingress-http           n2        UP
ingress-http           BACKEND   UP
ingress-https          FRONTEND  OPEN
ingress-https          cp1       DOWN    Connection refused
ingress-https          cp2       DOWN    Connection refused
ingress-https          cp3       DOWN    Connection refused
ingress-https          n1        UP
ingress-https          n2        UP
ingress-https          BACKEND   UP
api                    FRONTEND  OPEN
api                    cp1       UP
api                    cp2       UP
api                    cp3       UP
api                    BACKEND   UP
machine-config-server  FRONTEND  OPEN
machine-config-server  cp1       UP
machine-config-server  cp2       UP
machine-config-server  cp3       UP
machine-config-server  BACKEND   U
```

## Connect to OCP

`export KUBECONFIG=auth/kubeconfig`

`oc whoami --show-console`

`cat auth/kubeadmin-password`

`kubectl get nodes`


## Troubleshooting

Remove the Agent cache dir `rm -rf /home/rguske/.cache/agent`

`ssh -l core ocp1-cp1.rguske.coe.muc.redhat.com`