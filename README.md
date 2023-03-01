# openshift-disconnected-agent-installer

See [product docs](https://docs.openshift.com/container-platform/4.12/installing/installing_with_agent_based_installer/installing-with-agent-based-installer.html) for the supported steps and configurations.

See [agent based installer enhancement](https://github.com/dhellmann/openshift-enhancements/blob/ef85ba1660cfc6f9d56fe62ea20fb84d2e759918/enhancements/agent-installer/automated-workflow-for-agent-based-installer.md#steps-for-deploying-a-disconnected-cluster) for the PR details and steps described below.

## Prerequisites

So i want to follow the steps and documentation in a libvirt lab to install SNO using the agent installer.

Here's what i did for OpenShift 4.12 including my commands for a `Cluster.0` lab. 

I used a FC37 base VM host and a RHEL8 quay mirror host to install a SNO 4.12 cluster.

## Steps for Deploying a Disconnected Cluster

1. The infrastructure admin launches an image registry visible to the
   hosts that will form the cluster and to a host that has internet
   access so the release image can be downloaded.

   Grab the mirror registry binary from
   - https://console.redhat.com/openshift/downloads#tool-mirror-registry

   Install a rhel8 VM as the quay host (and install host) using libvirt and the kvm qcow2 image.

   ```bash
   # set a root password
   dnf -y install guestfs-tools
   virt-customize -a rhel-8.7-x86_64-kvm.qcow2 --root-password password:password
   # resize base image
   qemu-img resize rhel-8.7-x86_64-kvm.qcow2 +20G
   cp rhel-8.7-x86_64-kvm.qcow2 rhel-8.7-x86_64-kvm.qcow2.ORIGINAL
   virt-resize --expand /dev/vda3 rhel-8.7-x86_64-kvm.qcow2.ORIGINAL rhel-8.7-x86_64-kvm.qcow2
   rm rhel-8.7-x86_64-kvm.qcow2.ORIGINAL
   # use virt-install or virt-manager - 2 vCPU, 10GB ram and network
   # register the vm
   subscription-manager register --username=<user> --password=<password>
   ```

   Need to give it some extra disk for future releases and operator images etc.

   ```bash
   # on virt manager host
   lvcreate --virtualsize 300G -T fedora/thin_pool --name rhel8.7-lvm
   vgchange -ay -K fedora

   # in rhel8.7 vm add disk
   ssh root@192.168.86.23
   sudo
   mkdir /opt/quay
   chmod 777 /opt/quay
   mkfs.xfs /dev/vdb
   blkid
   echo "UUID=f5ba08b9-0e59-4531-ae0d-75351be56807 /opt/quay  xfs     defaults,nofail        0 0" >> /etc/fstab
   mount -a
   ```

   Copy quay mirror registry

   ```bash
   scp mirror-registry.tar.gz root@192.168.86.23:/opt/quay
   # untar it
   ssh root@192.168.86.23
   cd /opt/quay/
   tar xzvf mirror-registry.tar.gz
   image-archive.tar
   ```

   Install mirror registry. Ensure the hostname you use resolves in DNS, don't use localhost, else cannot push images later on.

   ```bash
   ./mirror-registry install \
     --quayHostname quay.eformat.me \
     --quayRoot /opt/quay/mirror-registry-root \
     --pgStorage /opt/quay/pg-data \
     --quayStorage /opt/quay/quay-storage \
     --sslCheckSkip \
     --initPassword password \
     --initUser admin \
     -v
   ```

   Quay should now be available on https://quay.eformat.me:8443

   Notes: You can also [create your own CA/cert](https://access.redhat.com/documentation/en-us/red_hat_quay/3/html/manage_red_hat_quay/using-ssl-to-protect-quay#create-a-ca-and-sign-a-certificate). You may also need to set up a proxy for mirror quay host if not directly connected to internet.

2. The cluster creator obtains the `oc` command line tool and uses `oc`
   to copy the OpenShift release image into the local image registry
   in one of the usual ways with [`oc adm
   mirror`](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-installation-images.html)
   or [the `oc mirror`
   plugin](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html).

   On the quay host get the `oc` client and `oc-mirror` plugin:

   ```bash
   wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.12/oc-mirror.tar.gz
   tar xzvf oc-mirror.tar.gz && chmod 755 oc-mirror
   mv oc-mirror /usr/local/bin/
   ```

   ```bash
   wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.12/openshift-client-linux.tar.gz
   tar xzvf openshift-client-linux.tar.gz && chmod 755 oc
   mv oc /usr/local/bin/
   ```

   Test it works ok.

   ```bash
   oc mirror help
   ```

   Login to RedHat and your quay instance

   ```bash
   cat /tmp/pull-secret | jq . > $XDG_RUNTIME_DIR/containers/auth.json
   podman login -u admin -p password quay.eformat.me:8443 --tls-verify=false
   ```

   Create a simple `ImageSetConfiguration` config (see the product docs for more examples). Note: if you also add operators, the mirroring can fail if they are not all packaged correctly for mirroring - start with just the install, then add operators in later phase.

   ```bash
   cat << EOF > /tmp/imageset.yaml
   kind: ImageSetConfiguration
   apiVersion: mirror.openshift.io/v1alpha2
   archiveSize: 4
   storageConfig:
   local:
       path: /opt/quay/ocp-mirror-offline
   mirror:
   platform:
       channels:
       - name: stable-4.12
       type: ocp
       graph: true
   operators:
   helm: {}
   EOF
   ```

   Create a disconnected file based mirror (rather than loading straight into our quay). This is in-case you really are fully disconnected. This will take some time.

   ```bash
   oc mirror --config=/tmp/imageset.yaml file:///opt/quay/ocp-mirror-offline -v3
   ```

   Load into our quay registry - this also creates `ImageContentSourcePolicy` and `CatalogSource` yaml resource.

   ```bash
   oc mirror --from=/opt/quay/ocp-mirror-offline docker://quay.eformat.me:8443 --dest-skip-tls -v3
   ```

   Note: if you wanted to generate the yaml resources only from your offline file repo:

   ```bash
   oc mirror --from=/opt/quay/ocp-mirror-offline docker://quay.eformat.me:8443 --dest-skip-tls --manifests-only
   ```

   You should see these files generated:

   ```bash
   ls oc-mirror-workspace/results-1677533027
   imageContentSourcePolicy.yaml  mapping.txt
   ```

3. The cluster creator obtains the OpenShift installer in one of [the
   usual
   ways](https://docs.openshift.com/container-platform/4.12/installing/installing_bare_metal/installing-bare-metal.html#installation-obtaining-installer_installing-bare-metal).

   I Just download it (but you can extract the version from oc).

   ```bash
   OPENSHIFT_VERSION=4.12.4
   curl --write-out "%{http_code}" https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/${OPENSHIFT_VERSION}/openshift-install-linux.tar.gz -o openshift-install-linux.tar.gz
   tar xzvf openshift-install-linux.tar.gz
   chmod 755 openshift-install
   ```

4. The cluster creator triggers their orchestration tool to run the
   installer.

   One step here, is for the vm install host, ensure you have `nmstate` installed

   ```bash
   sudo dnf install /usr/bin/nmstatectl -y
   ```

   I also use libvirt to create the virtual network.

   ```xml
   cat <<EOF > /etc/libvirt/qemu/networks/baz.xml
   <network>
   <name>baz</name>
   <uuid>fc23191f-de21-4bf5-774b-98711b9f3d9e</uuid>
   <forward mode='nat' size='1500'/>
   <bridge name='tt0' stp='on' delay='0'/>
   <mac address='52:54:00:22:4d:3a'/>
   <domain name='eformat.me'/>
   <dns enable='yes'>
       <host ip='192.168.130.10'>
       <hostname>api.baz.eformat.me</hostname>
       <hostname>api-int.baz.eformat.me</hostname>
       <hostname>console-openshift-console.apps.baz.eformat.me</hostname>
       <hostname>oauth-openshift.apps.baz.eformat.me</hostname>
       <hostname>canary-openshift-ingress-canary.apps.baz.eformat.me</hostname>
       </host>
   </dns>
   <ip family='ipv4' address='192.168.130.1' netmask='255.255.255.0'>
       <dhcp>
       <range start='192.168.130.20' end='192.168.130.254'/>
       <host mac='52:54:00:22:4d:4a' name='baz' ip='192.168.130.10'/>
       </dhcp>
   </ip>
   </network>
   EOF
   ```

   Create network.

   ```bash
   virsh net-define /etc/libvirt/qemu/networks/baz.xml
   virsh net-start baz
   virsh net-autostart baz
   # just in case you need to delete it!
   virsh net-destroy baz
   virsh net-undefine baz
   # dns mask
   echo server=/api.baz.eformat.me/192.168.130.1 | sudo tee /etc/NetworkManager/dnsmasq.d/openshift.conf
   ```

5. The orchestration tool (or cluster creator) prepares an
   `install-config.yaml`, including `NMState` content to configure the
   static network settings for the hosts.

   I am creating a cluster called `baz`. Put input artifacts in a folder.

   ```bash
   mkdir ~/tmp/baz && cd ~/tmp/baz
   ```

   Create your `install-config.yaml` file. This is a simple SNO cluster. Add your `quay pull secret`, Your public `sshKey`, and the contents from `imageContentSourcePolicy.yaml` to `imageContentSources`. You will need the CA from your quay install (mine was in `/opt/quay/mirror-registry-root/quay-rootCA/` folder) in the `additionalTrustBundle`.

   ```yaml
   cat << EOF > install-config.yaml
   apiVersion: v1
   baseDomain: eformat.me
   metadata:
     name: baz
   networking:
     networkType: OVNKubernetes
     machineNetwork:
     - cidr: 192.168.130.0/24
   compute:
   - name: worker
     replicas: 0
   controlPlane:
     architecture: amd64
     name: master
     replicas: 1
   platform:
     none: {}
   pullSecret: '{"auths":{"quay.eformat.me:8443":{"auth":"..."}}}'
   sshKey: |
     ssh-rsa AAAAB...
   imageContentSources:
   - mirrors:
     - quay.eformat.me:8443/openshift/release
     source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
   - mirrors:
     - quay.eformat.me:8443/openshift/release-images
     source: quay.io/openshift-release-dev/ocp-release
   - mirrors:
     - quay.eformat.me:8443/ubi8
     source: registry.access.redhat.com/ubi8
   additionalTrustBundle: |
     -----BEGIN CERTIFICATE-----
     MIID ...
     -----END CERTIFICATE-----
   EOF
   ```

   This is the agent file. You need to make sure all of the `nmstate` details are correct for your network. I also add in a static route to find the quay registry and ensure the `rendezvousIP` and `mac-address` matches the libvirt network i created above.

   ```bash
   cat > agent-config.yaml << EOF
   apiVersion: v1alpha1
   kind: AgentConfig
   metadata:
   name: baz
   rendezvousIP: 192.168.130.10
   hosts: 
   - hostname: master-0
       interfaces:
       - name: enp1s0
           macAddress: 52:54:00:22:4d:4a
       rootDeviceHints: 
       deviceName: /dev/vda
       networkConfig: 
       interfaces:
           - name: enp1s0
           type: ethernet
           state: up
           mac-address: 52:54:00:22:4d:4a
           ipv4:
               enabled: true
               address:
               - ip: 192.168.130.10
                   prefix-length: 24
               dhcp: false
       dns-resolver:
           config:
           server:
               - 192.168.130.1
       routes:
           config:
           - destination: 0.0.0.0/0
               next-hop-address: 192.168.130.1
               next-hop-interface: enp1s0
               table-id: 254
           - destination: 192.168.86.0/0
               next-hop-address: 192.168.130.1
               next-hop-interface: enp1s0
               table-id: 254
   EOF
   ```

   Finally copy these into a `cluster` folder.

   ```bash
   mkdir cluster
   cp install-config.yaml agent-config.yaml cluster/
   ```

   Note: Disconnected NTP servers may also need to be configured in the [agent config](https://github.com/openshift/installer/blob/master/pkg/asset/agent/agentconfig/agent_config.go#L60-L62) else defaults to:

   ```yaml
    additionalNTPSources:
    - 0.rhel.pool.ntp.org
    - 1.rhel.pool.ntp.org
   ```

6. The orchestration tool runs `openshift-install agent create image`.

   Run:

   ```bash
   ./openshift-install --dir cluster agent create image
   ```

7. The installer extracts a copy of the RHCOS ISO from the OpenShift
   release payload.
8. The installer combines the inputs provided by the user with
   generated information like the InfraEnv ID and REST API credentials
   to produce an Ignition config.
9. The installer combines the Ignition config and the RHCOS ISO to
   create the bootable image configured to run the assisted service and/or
   agent based on the host that boots the image and writes the new
   image and cluster credentials to disk.
10. The orchestration tool copies the ISO to a location that the hosts
    or VMs can boot from it. For bare metal, that will generally be a
    web server visible to the management controllers in the bare metal
    hosts (because the management controllers are unlikely to be on
    the same network as the external interfaces for the cluster
    hosts). For vSphere or other hypervisors, that may be a filesystem
    on the hypervisor host.

    Copy the generated `agent.x86_64.iso` ISO so we can virt-install it.

    ```bash
    sudo cp cluster/agent.x86_64.iso /var/lib/libvirt/images/agent.x86_64.iso
    sudo chown qemu:qemu /var/lib/libvirt/images/agent.x86_64.iso
    sudo restorecon -rv /var/lib/libvirt/images/agent.x86_64.iso
    ```

11. The orchestration tool configures the host to boot the ISO. For
    bare metal, that means configuring the baseboard management
    controller (BMC) in each host to expose the ISO through the
    virtual media interface. For vSphere or other hypervisors, similar
    settings of the VM need to be adjusted.
12. The orchestration tool boots the hosts.

   Create a thin lvm for the cluster install disk.

   ```bash
   lvcreate --virtualsize 120G --name baz -T fedora/thin_pool
   vgchange -ay -K fedora
   ```

   Boot the SNO host. You will need these minimum specs:

   ```bash
   VM_NAME=baz
   NET_NAME=baz
   OS_VARIANT="fedora-coreos-stable"
   RAM_MB="16384"
   DISK_GB="120"
   CPU_CORE="8"
   RHCOS_ISO=/var/lib/libvirt/images/agent.x86_64.iso
   MAC=52:54:00:22:4d:4a
   ```

   Adjust to suit.

   ```bash
   rm -f nohup.out
   nohup virt-install \
       --virt-type kvm \
       --connect qemu:///system \
       -n "${VM_NAME}" \
       -r "${RAM_MB}" \
       --vcpus "${CPU_CORE}" \
       --os-variant="${OS_VARIANT}" \
       --cpu=host-passthrough,cache.mode=passthrough \
       --import \
       --network=network:${NET_NAME},mac=${MAC},driver.queues=4 \
       --events on_reboot=restart \
       --cdrom "${RHCOS_ISO}" \
       --disk path=/dev/fedora/baz,io=io_uring,cache='writeback',discard='unmap' \
       --boot hd,cdrom \
       --wait=-1 &
   ```

13. The orchestration tool uses `openshift-install agent wait-for install-complete` to
    watch the deployment progress and wait for it to complete.

   ```bash
   cd ~/tmp/baz
   ./openshift-install --dir cluster agent wait-for bootstrap-complete --log-level=debug
   ```

   If this is the first time you tried agent install, you will need these helpful tips to debug anything that goes wrong.

   ```bash
   ssh core@192.168.130.10
   sudo su -
   journactl -f
   systemctl status agent.service
   systemctl status assisted-service.service
   journalctl -u assisted-service.service
   podman ps
   ```

   Any fatal errors you see in the journal will need troubleshooting. It took several attempts until i got my configuration just right for a successful install.

14. On each host, the startup script in the image selects the correct
    network settings based on the MAC addresses visible and applies
    the network settings for each interface.
15. On node0, the startup script in the image recognizes that it is on
    node0 and launches the assisted service, passing generated
    one-time-use data such as the InfraEnv ID and REST API
    credentials. It also starts the create-cluster service to drive
    the deployment.
16. On node0, the create-cluster service waits for the assisted
    service to start, then creates an InfraEnv using the API.
17. In parallel
    - On node0, the create-cluster service waits for the number of
      agents registered with the service to match the expected value
      based on the input from the user.
    - On all nodes (including node0), the startup script in the image
      launches the assisted agent with the InfraEnv ID and configured
      to connect to the assisted service on node0.
18. All of the agents register their hosts with the assisted service.
19. The assisted service performs its standard validations (network
    connectivity, storage space, RAM, etc.) for all of the hosts.
20. On node0, the create-cluster service sees that all known agents
    have registered and the hosts have passed validation. It uses the
    assisted service API to trigger the deployment to proceed.
21. The assisted service performs the cluster deployment in its usual
    way, except that the nodes will use the ISOs they booted from to
    install RHCOS instead of fetching a new image from the assisted
    service.
22. The `openshift-install agent wait-for install-complete` process sees the OpenShift
    API become available and starts using it to watch the cluster
    complete deployment, combining the information provided by
    `ClusterVersion` and `ClusterOperator` resources with the assisted
    service progress API.
23. On node0, when all of the other hosts have been deployed and
    bootstrapping is complete, the agent reboots node0.
24. node0 boots from its internal storage and joins the cluster as a
    control plane node.
25. The `openshift-install agent wait-for install-complete` process loses
    the connection to the assisted service REST API and starts relying
    entirely on the OpenShift API for data.
26. The `openshift-install agent wait-for install-complete` process sees that the
    `ClusterVersion` API in the cluster shows that the deployment has
    completed, and reports success then exits.
27. The orchestration tool may take steps to clean up (removing the
    ISO, disconnecting it from the boot sequence for the VMs or BMCs,
    etc.).

   A successful install will end in the familiar:

   ```bash
   openshift-install --dir cluster agent wait-for install-complete --log-level=debug
   ```

   ```bash
   INFO Cluster is installed
   INFO Install complete!
   INFO To access the cluster as the system:admin user when using 'oc', run 
   INFO     export KUBECONFIG=/home/mike/tmp/baz/cluster/auth/kubeconfig 
   INFO Access the OpenShift web-console here: https://console-openshift-console.apps.baz.eformat.me 
   INFO Login to the console with user: "kubeadmin", and password: "..."
   ```
