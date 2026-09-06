**Background:**

AAP 2.7 introduces a new Automation Orchestrator (AO) feature, which requires an OpenShift-based architecture (deployed via Operator). This means that if you previously installed AAP using the Container Base method, you would now need to separately deploy an OCP cluster and then install the AO Operator on top of it. For customers in China, this is highly impractical — whether for POC purposes or real-world production scenarios.

A more viable alternative is to use MicroShift, a lightweight OCP environment provided by Red Hat for edge use cases. MicroShift can be thought of as a lightweight, stripped-down OCP cluster that runs on a single node. The idea is to use MicroShift solely as the base platform, and then install both AAP and AO on it via Operators.

This approach satisfies the deployment requirements of both AAP and AO while preserving the single-node architecture used before, without generating excessive resource overhead. However, adopting this approach means a fundamental change in how AAP is deployed — it introduces a large number of additional deployment and configuration steps. This can be particularly challenging in environments (as is common for most customers) where outbound internet access is not available.

This document aims to provide as complete a record as possible of the installation and deployment of MicroShift, AAP, AO, MCP, and other common components. Optimization and real-world problem-solving will be addressed in later stages of actual use.

**Environment Preparation:**

- RHEL 9.6 virtual machine: 8 vCPUs, 32 GB RAM, 80 GB system disk + 100 GB additional disk;

- Disable and turn off both the firewall and SELinux;

- Update `/etc/hosts` to add the mapping between the IP address and the host FQDN;

- Create a regular user `admin` and update `/etc/sudoers` to allow passwordless privilege escalation to root;

- After a default RHEL 9.6 installation, the root partition resides in the `rhel` volume group. Use `vgscan` to extend the `rhel` VG with the additional disk, but do **not** extend the root partition or create a new logical volume on it;

(MicroShift uses the Logical Volume Manager Storage (LVMS) Container Storage Interface (CSI) plugin to provide storage for Persistent Volumes (PVs). LVMS relies on the Linux Logical Volume Manager (LVM) to dynamically manage the underlying Logical Volumes (LVs) for PVs. Therefore, your machine must have an LVM Volume Group (VG) with unused space, from which LVMS can create the LVs needed for workload PVs.

To configure a VG that allows LVMS to create LVs for workload PVs, reduce the "required size" of the root volume during RHEL installation. Reducing the root volume size leaves unallocated disk space for LVMS to create additional LVs at runtime.)

```
[root@localhost ~]# hostnamectl set-hostname microshift.example.com

[root@localhost ~]# pvcreate /dev/sdb
Physical volume "/dev/sdb" successfully created.

[root@localhost ~]# vgscan
Found volume group "rhel" using metadata type lvm2

[root@localhost ~]# vgextend rhel /dev/sdb
Volume group "rhel" successfully extended
```

- Register the system to RHN, enable the required repositories, and lock the OS version:

```
[root@microshift ~]# subscription-manager register --auto-attach

[root@microshift ~]# subscription-manager repos --enable rhocp-4.22-for-rhel-9-$(uname -m)-rpms --enable fast-datapath-for-rhel-9-$(uname -m)-rpms

[root@microshift ~]# subscription-manager release --set=9.6
```

- Download the MicroShift and `oc` packages:

```
[root@microshift ~]# yum install -y microshift openshift-clients
```

- Obtain the pull secret:

Log in to the Red Hat Hybrid Cloud Console (console.redhat.com) to download the Pull Secret. Save the downloaded file on the local machine (`~/pull-secret.txt`) and configure it (run the following as the `admin` user):

```
[admin@microshift ~]$ cat pull-secret.txt 
{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfY2VlOGRmODk0Nzc2NGM3NWE5NjI2NjNmYjI1YjVlZmQ6U1NTMU8xM1ZMVEdOWVM4VElHWkNYS1ZIWUlYSEQ3UzBONldDWDdUSjlRWkEzMzBRN0g4UFE0SEoxSjJNU1lKWQ==","email":"jewang@redhat.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfY2VlOGRmODk0Nzc2NGM3NWE5NjI2NjNmYjI1YjVlZmQ6U1NTMU8xM1ZMVEdOWVM4VElHWkNYS1ZIWUlYSEQ3UzBONldDWDdUSjlRWkEzMzBRN0g4UFE0SEoxSjJNU1lKWQ==","email":"jewang@redhat.com"},"registry.connect.redhat.com":{"auth":"NTM0MjM3MTN8dWhjLTFlckN4dzg1N0ptbVJQbTNLVmlsVG5JUmF4TjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmpZVGc0WXpaaU1qWTNPREEwT0RKbVlUVTNNRGd3WkdZd01tSTFNREEyTWlKOS5TQUhvLTRVTFlkM05qQlV6U0dSR2xSV3ZTZGE1Tk1CcmcybUVueWQ4bHgyaWhGZHFGejdfZFZ3MWRhbXRwZjdGTzlNV2dZNDhwTjhPSkJUdGV6aElIOTV6akY0RDRRNHZOZHBNckI0V0hlUkY2Qmh6WWg4aWdDVmpGVjdKQzY3cTBKZ3cya3NMcU82U0R6VVRBOXdyTnNZdmhOOGMtUWxOSFl5T2N5WDBvajZzNW43Y1VacVlIcVg4RzlyS3BSSWVnY1hITXVLS2dxRU9fS18ycVBET1RPYkNtMU5JZFR0Vm5wbUltRDA1ZUdEZGdGbzJSbjdVajcxcDZVQ0NNeHJkbGxucEY4T3hSeVp5bU5KMlpPQ0NFNWxoZGRZa2luMGRSU2ZkckFEaExILWVUMGtBaXFrTC1DdVFqRTR0NzdRMjhqUkFjcXlFNkIzWU84am5vVkhEZ1VnQnp6UFk1aGY0eXNQc1V3Y08yRWM0aGsyLXhKZXNUUm9PVG5Bd2t3cFJRNHBCOFVmb3lBZFJJeTMzUTZNUGwwXzFHblRLZFFvb3V3WHJ1SW56bEhlVHpDck9XNno2S3lCTDRhYlZyQnFVam1yMktQcWJmcFk3TUkyUHFjbTdNLU9fcU1QMEctXzhzUDJDcjdfZ0N1Z2JJVUFkSHpXQXA2WXZwMXFrVFVNa2lFdzFmWkNHTXFkaXFmMzd3WUJwamlFT29uVkU4MkVTdEl0aE00cnBtUXRBQi1rSU1nUUNfTEtYTVZBT2VRM0owa3RWUHJjc1VuamtZZWU4OGZUVGI1Q0c2NjJjODJNaTZldHhuektXRnQxTG1QWFRna3hfRnZnX2lacEkyUVFPMWxXcm12WjZxWWZMNUdWMWlHUFRQekRPeGhDOGZXbnlYaE9DWlhzMWR5QQ==","email":"jewang@redhat.com"},"registry.redhat.io":{"auth":"NTM0MjM3MTN8dWhjLTFlckN4dzg1N0ptbVJQbTNLVmlsVG5JUmF4TjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSmpZVGc0WXpaaU1qWTNPREEwT0RKbVlUVTNNRGd3WkdZd01tSTFNREEyTWlKOS5TQUhvLTRVTFlkM05qQlV6U0dSR2xSV3ZTZGE1Tk1CcmcybUVueWQ4bHgyaWhGZHFGejdfZFZ3MWRhbXRwZjdGTzlNV2dZNDhwTjhPSkJUdGV6aElIOTV6akY0RDRRNHZOZHBNckI0V0hlUkY2Qmh6WWg4aWdDVmpGVjdKQzY3cTBKZ3cya3NMcU82U0R6VVRBOXdyTnNZdmhOOGMtUWxOSFl5T2N5WDBvajZzNW43Y1VacVlIcVg4RzlyS3BSSWVnY1hITXVLS2dxRU9fS18ycVBET1RPYkNtMU5JZFR0Vm5wbUltRDA1ZUdEZGdGbzJSbjdVajcxcDZVQ0NNeHJkbGxucEY4T3hSeVp5bU5KMlpPQ0NFNWxoZGRZa2luMGRSU2ZkckFEaExILWVUMGtBaXFrTC1DdVFqRTR0NzdRMjhqUkFjcXlFNkIzWU84am5vVkhEZ1VnQnp6UFk1aGY0eXNQc1V3Y08yRWM0aGsyLXhKZXNUUm9PVG5Bd2t3cFJRNHBCOFVmb3lBZFJJeTMzUTZNUGwwXzFHblRLZFFvb3V3WHJ1SW56bEhlVHpDck9XNno2S3lCTDRhYlZyQnFVam1yMktQcWJmcFk3TUkyUHFjbTdNLU9fcU1QMEctXzhzUDJDcjdfZ0N1Z2JJVUFkSHpXQXA2WXZwMXFrVFVNa2lFdzFmWkNHTXFkaXFmMzd3WUJwamlFT29uVkU4MkVTdEl0aE00cnBtUXRBQi1rSU1nUUNfTEtYTVZBT2VRM0owa3RWUHJjc1VuamtZZWU4OGZUVGI1Q0c2NjJjODJNaTZldHhuektXRnQxTG1QWFRna3hfRnZnX2lacEkyUVFPMWxXcm12WjZxWWZMNUdWMWlHUFRQekRPeGhDOGZXbnlYaE9DWlhzMWR5QQ==","email":"jewang@redhat.com"}}}

```

```
[root@microshift ~]$ sudo cp ~/pull-secret.txt /etc/crio/openshift-pull-secret

[root@microshift ~]$ sudo chown admin:admin /etc/crio/openshift-pull-secret
[root@microshift ~]$ sudo chmod 644 /etc/crio/openshift-pull-secret
```

- In a China network environment, configure the MicroShift service to pull images via a proxy:

```
[root@microshift ~]$ sudo mkdir -p /etc/systemd/system/crio.service.d

[root@microshift ~]$ cat <<EOF | sudo tee /etc/systemd/system/crio.service.d/HTTP-PROXY.conf
[Service]
Environment="HTTP_PROXY=http://10.210.65.9:10000"
Environment="HTTPS_PROXY=http://10.210.65.9:10000"
Environment="NO_PROXY=localhost,127.0.0.1,.cluster.local,.svc,10.42.0.0/16,169.254.169.1,microshift.example.com"
EOF
```

- Create a proxy script under `/etc/profile.d/` to ensure it takes effect for all users and system services at login:

```
[root@microshift ~]$ cat <<'EOF' | sudo tee /etc/profile.d/proxy.sh
export http_proxy="http://10.210.65.9:10000"
export https_proxy="http://10.210.65.9:10000"
export HTTP_PROXY="http://10.210.65.9:10000"
export HTTPS_PROXY="http://10.210.65.9:10000"
export no_proxy="localhost,127.0.0.1,.cluster.local,.svc,10.42.0.0/16,169.254.169.1,10.210.65.8,microshift.example.com"
export NO_PROXY="localhost,127.0.0.1,.cluster.local,.svc,10.42.0.0/16,169.254.169.1,10.210.65.8,microshift.example.com"
EOF


[root@microshift ~]$ sudo chmod +x /etc/profile.d/proxy.sh
[root@microshift ~]$ source /etc/profile.d/proxy.sh
```

**Installation & Deployment (1) — Deploy MicroShift and OLM:**

- Reload and restart services:

```
[root@microshift ~]$ sudo systemctl daemon-reload
[root@microshift ~]$ sudo systemctl restart crio
[root@microshift ~]$ sudo systemctl enable crio --now
[root@microshift ~]$ sudo systemctl enable microshift --now
```

- Configure `oc` / `kubectl` CLI access:

```
[root@microshift ~]$ mkdir -p ~/.kube

[root@microshift ~]$ sudo cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
[root@microshift ~]$ chmod 600 ~/.kube/config
```

- During MicroShift startup, the required Pods will be launched. Verify with the following commands:

```
[admin@microshift ~]$ oc get nodes
[admin@microshift ~]$ oc get pods -A

```

Example output:

```
[admin@microshift ~]$ oc get nodes
NAME                     STATUS   ROLES                         AGE    VERSION
microshift.example.com   Ready    control-plane,master,worker   4m4s   v1.35.6


[admin@microshift ~]$ oc get pods -A 
NAMESPACE                  NAME                                       READY   STATUS    RESTARTS   AGE
kube-system                csi-snapshot-controller-5b6dcfcc77-5xwrp   1/1     Running   0          3m21s
openshift-dns              dns-default-qhdqd                          2/2     Running   0          110s
openshift-dns              node-resolver-q2nbv                        1/1     Running   0          3m17s
openshift-ingress          router-default-67dcc99b4f-vx5bb            1/1     Running   0          3m18s
openshift-ovn-kubernetes   ovnkube-master-9pxsj                       5/5     Running   0          3m17s
openshift-ovn-kubernetes   ovnkube-node-t266p                         1/1     Running   0          3m17s
openshift-service-ca       service-ca-7cd8fbb87-hx2l5                 1/1     Running   0          3m21s
openshift-storage          lvms-operator-5fbfcd5b9-zwcht              1/1     Running   0          3m19s
openshift-storage          vg-manager-8r8mt                           1/1     Running   0          20s

```

**Note:**

If any Pods are in Pending state, use `kubectl describe` to inspect the specific events. Since MicroShift is a single-node cluster, the node may sometimes be Tainted for various reasons, preventing Pod scheduling. For example:

```
Node-Selectors: node-role.kubernetes.io/master=
Events: 0/1 nodes are available: 1 node(s) had untolerated taint(s).
```

In MicroShift, the three most common causes of CNI/OVN network plugin hangs are:

- The system firewall is blocking the OVN bridge
- The CRIO network interface has not been initialized
- Hostname resolution mismatch

To remove Taints, use the following approach:

```
# 1. Get node name
oc get nodes

# 2. Check node Taints and Labels (replace <node-name> with the actual node name, e.g. microshift.demo.local)
oc describe node <node-name> | grep -iE "taints|labels" -A 5
```

If the output shows taints like `node-role.kubernetes.io/master:NoSchedule` or `node.kubernetes.io/not-ready`, run the following to remove them:

```
# Remove all common taints that may block scheduling (trailing - means delete the taint)
oc taint nodes --all node-role.kubernetes.io/master:NoSchedule- 2>/dev/null || true
oc taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule- 2>/dev/null || true
oc taint nodes --all node.kubernetes.io/unschedulable- 2>/dev/null || true
```

Label the node with control-plane roles (MicroShift system Pods rely on the `node-role.kubernetes.io/master=` node selector):

```
# Add master and control-plane role labels to the node
oc label node --all node-role.kubernetes.io/master="" --overwrite
oc label node --all node-role.kubernetes.io/control-plane="" --overwrite
```

Check network interface and CRIO service (for ContainerCreating hangs)

Since `ovnkube-master` is also stuck in ContainerCreating, the network plugin has not yet started. Check the host's CRIO status and firewall to ensure internal networking is not blocked:

```
# 1. Ensure SELinux or firewall is not blocking
sudo systemctl restart crio
sudo systemctl restart microshift

# 2. Watch Pod status in real time to see if they transition to Running
oc get pods -A -w
```

- Install OLM (MicroShift does not include OLM — Operator Lifecycle Management — by default):

```
[admin@microshift ~]$ export KUBECONFIG=/home/admin/.kube/config
[admin@microshift ~]$ oc get nodes
[admin@microshift ~]$ oc get pods -A | head -50

[admin@microshift ~]$ sudo yum install microshift-olm
```

- Restart MicroShift:

```
[admin@microshift ~]$ sudo -n systemctl restart microshift
[admin@microshift ~]$ sudo -n systemctl status microshift --no-pager
```

- Verify the OLM namespace and controllers:

```
[admin@microshift ~]$ oc get ns
[admin@microshift ~]$ oc get pods -n openshift-operator-lifecycle-manager
```

Confirm the following NS and Pods are running normally:

  - (NS) openshift-operator-lifecycle-manager

  - (Pod) olm-operator

  - (Pod) catalog-operator

```
[admin@microshift ~]$ oc get ns
NAME                                   STATUS   AGE
default                                Active   8m25s
kube-node-lease                        Active   8m25s
kube-public                            Active   8m25s
kube-system                            Active   8m25s
openshift-controller-manager           Active   8m17s
openshift-dns                          Active   8m9s
openshift-infra                        Active   8m20s
openshift-ingress                      Active   8m10s
openshift-kube-controller-manager      Active   8m20s
openshift-marketplace                  Active   65s
openshift-operator-lifecycle-manager   Active   65s
openshift-operators                    Active   65s
openshift-ovn-kubernetes               Active   8m9s
openshift-route-controller-manager     Active   8m17s
openshift-service-ca                   Active   8m12s
openshift-storage                      Active   8m10s

[admin@microshift ~]$ oc get pods -n openshift-operator-lifecycle-manager
NAME                                READY   STATUS    RESTARTS   AGE
catalog-operator-744c68ff46-p2lk9   1/1     Running   0          5m55s
olm-operator-69499fd5c7-9kt4v       1/1     Running   0          5m55s
```

- Configure kubeconfig:

The default MicroShift kubeconfig is located at `/var/lib/microshift/resources/kubeadmin/kubeconfig`. If permissions are incorrect, copy it to the `admin` account and verify:

```
[admin@microshift ~]$ mkdir -p /home/admin/.kube
[admin@microshift ~]$ cp /var/lib/microshift/resources/kubeadmin/kubeconfig /home/admin/.kube/config
[admin@microshift ~]$ chown -R admin:admin /home/admin/.kube
[admin@microshift ~]$ chmod 600 /home/admin/.kube/config

[admin@microshift ~]$ export KUBECONFIG=/home/admin/.kube/config
[admin@microshift ~]$ oc get nodes
```

- Configure Red Hat Operator pull secret:

```
[admin@microshift ~]$ cat /etc/crio/openshift-pull-secret
```

- Create the secret:

```
[admin@microshift ~]$ kubectl create secret generic redhat-pull-secret \
  -n openshift-marketplace \
  --from-file=.dockerconfigjson=/etc/crio/openshift-pull-secret \
  --type=kubernetes.io/dockerconfigjson
```

Sometimes this secret needs to be associated with the CatalogSource; otherwise the Operator will not appear.

- Register the Red Hat CatalogSource with OLM

Create a CatalogSource:

```
[admin@microshift ~]$ cat catalogsource.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: registry.redhat.io/redhat/redhat-operator-index:v4.22
  displayName: Red Hat Operators
  priority: 100
  publisher: Red Hat
  secrets:
    - redhat-operator-pull-secret
    
[admin@microshift ~]$ oc apply -f catalogsource.yaml
```

Verify with:

```
[admin@microshift ~]$ oc get catalogsource -A
NAMESPACE               NAME               DISPLAY             TYPE   PUBLISHER   AGE
openshift-marketplace   redhat-operators   Red Hat Operators   grpc   Red Hat     7m3s

[admin@microshift ~]$ oc get pods -n openshift-marketplace
NAME                     READY   STATUS    RESTARTS        AGE
redhat-operators-zxs7c   1/1     Running   1 (4m15s ago)   7m16s

```

If the Red Hat Operator catalog does not appear, check:

  - Whether the pull secret is correct

  - Whether `registry.redhat.io` is accessible

  - Whether `catalog-operator` / `olm-operator` are running normally

- Create the namespace and OperatorGroup:

```
[admin@microshift ~]$ oc create ns ansible-automation-platform
```

- Create OperatorGroup:

```
[admin@microshift ~]$ cat operatorgroup.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: aap-operator-group
  namespace: ansible-automation-platform
spec:
  targetNamespaces:
    - ansible-automation-platform
    
[admin@microshift ~]$ oc apply -f operatorgroup.yaml
```

**Installation & Deployment (2) — Deploy the AAP Operator:**

- Install the AAP Operator:

```
[admin@microshift ~]$ cat subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: aap-operator-subscription
  namespace: ansible-automation-platform
spec:
  channel: stable-2.7
  installPlanApproval: Automatic
  name: ansible-automation-platform-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  
[admin@microshift ~]$ oc apply -f subscription.yaml
```

- Verify the CSV and sub-operators:

```
[admin@microshift ~]$ oc get csv -n ansible-automation-platform
[admin@microshift ~]$ oc get sub -n ansible-automation-platform
[admin@microshift ~]$ oc get pods -n ansible-automation-platform
```

Once the AAP Operator is complete, 7 sub-operators should appear:

gateway, lightspeed, automation-controller, automation-hub, metrics, eda-server, resource

```
[admin@microshift ~]$ oc get csv -n ansible-automation-platform
NAME                               DISPLAY                       VERSION              RELEASE   REPLACES                           PHASE
aap-operator.v2.7.0-0.1787240540   Ansible Automation Platform   2.7.0+0.1787240540             aap-operator.v2.7.0-0.1785438985   Succeeded

[admin@microshift ~]$ oc get sub -n ansible-automation-platform
NAME                        PACKAGE                                SOURCE             CHANNEL
aap-operator-subscription   ansible-automation-platform-operator   redhat-operators   stable-2.7

[admin@microshift ~]$ oc get pods -n ansible-automation-platform -w
NAME                                                              READY   STATUS    RESTARTS   AGE
aap-gateway-operator-controller-manager-7587f9dc95-8cc4f          1/1     Running   0          4m58s
ansible-lightspeed-operator-controller-manager-5779bb879b-b2znn   1/1     Running   0          4m57s
automation-controller-operator-controller-manager-7cb4d5d5lwztx   1/1     Running   0          4m58s
automation-hub-operator-controller-manager-7fdd649879-sww5x       1/1     Running   0          4m57s
automationmetricsservice-operator-controller-manager-779f4gf9hr   1/1     Running   0          4m57s
eda-server-operator-controller-manager-58fdcfc895-px9qz           1/1     Running   0          4m58s
resource-operator-controller-manager-fc85f696d-c7t4t              1/1     Running   0          4m57s

```

- Create the Automation Controller:

Create the resource:

```
[admin@microshift ~]$ cat automation-controller.yaml
apiVersion: automationcontroller.ansible.com/v1beta1
kind: AutomationController
metadata:
  name: automation-controller
  namespace: ansible-automation-platform
spec:
  admin_user: admin
  admin_password_secret: automation-controller-admin-password
  
[admin@microshift ~]$ oc apply -f automation-controller.yaml
```

Create the password and verify:

```
[admin@microshift ~]$ oc create secret generic automation-controller-admin-password \
  --from-literal=password=redhat \
  -n ansible-automation-platform
  
[admin@microshift ~]$ oc get pods -n ansible-automation-platform | grep automation-controller
[admin@microshift ~]$ oc get pvc -n ansible-automation-platform
```

Once successful, the following Pods will be running:

  - web 3/3 Running

  - task 4/4 Running

  - postgres started normally

  - Controller Web UI is accessible

```
[admin@microshift ~]$ oc get pvc -n ansible-automation-platform
NAME                                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          VOLUMEATTRIBUTESCLASS   AGE
postgres-15-automation-controller-postgres-15-0   Bound    pvc-9a9ec88a-6e51-4147-9300-f8f917abac1e   8Gi        RWO            topolvm-provisioner   <unset>                 10m

[admin@microshift ~]$ oc get pods -n ansible-automation-platform 
NAME                                                              READY   STATUS      RESTARTS      AGE
aap-gateway-operator-controller-manager-7587f9dc95-8cc4f          1/1     Running     1 (12m ago)   19m
ansible-lightspeed-operator-controller-manager-5779bb879b-b2znn   1/1     Running     1 (12m ago)   19m
automation-controller-migration-4.8.6-2kjhw                       0/1     Completed   0             7m28s
automation-controller-operator-controller-manager-7cb4d5d5lwztx   1/1     Running     0             19m
automation-controller-postgres-15-0                               1/1     Running     0             10m
automation-controller-task-74f99bd647-x7r8d                       4/4     Running     0             9m25s
automation-controller-web-5778968897-4bvfx                        3/3     Running     0             9m27s
automation-hub-operator-controller-manager-7fdd649879-sww5x       1/1     Running     0             19m
automationmetricsservice-operator-controller-manager-779f4gf9hr   1/1     Running     1 (12m ago)   19m
eda-server-operator-controller-manager-58fdcfc895-px9qz           1/1     Running     1 (12m ago)   19m
resource-operator-controller-manager-fc85f696d-c7t4t              1/1     Running     1 (12m ago)   19m

```

- Create the AnsibleAutomationPlatform CR:

At this point, the AAP Gateway operator is running, but no `AnsibleAutomationPlatform` CR exists, so the Platform Gateway is not deployed and no Route is created.

Create the `AnsibleAutomationPlatform` CR (Custom Resource) with:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatform
metadata:
  name: aap
  namespace: ansible-automation-platform
spec:
  controller:
    name: automation-controller
  hub:
    disabled: true
  eda:
    disabled: true
  lightspeed:
    disabled: true
EOF

```

Once complete, Pod status will change:

```
[admin@microshift ~]$ kubectl get pods -n ansible-automation-platform
NAME                                                              READY   STATUS      RESTARTS      AGE
aap-gateway-85f8c95b9-djzg6                                       2/2     Running     0             3m42s
aap-gateway-operator-controller-manager-7587f9dc95-8cc4f          1/1     Running     2 (62m ago)   82m
aap-postgres-15-0                                                 1/1     Running     0             4m44s
aap-redis-0                                                       1/1     Running     0             4m59s
ansible-lightspeed-operator-controller-manager-5779bb879b-b2znn   1/1     Running     2 (62m ago)   82m
automation-controller-migration-4.8.6-2kjhw                       0/1     Completed   0             70m
automation-controller-operator-controller-manager-7cb4d5d5lwztx   1/1     Running     1 (62m ago)   82m
automation-controller-postgres-15-0                               1/1     Running     0             73m
automation-controller-task-74f99bd647-x7r8d                       4/4     Running     0             72m
automation-controller-web-5778968897-4bvfx                        3/3     Running     0             72m
automation-hub-operator-controller-manager-7fdd649879-sww5x       1/1     Running     1 (62m ago)   82m
automationmetricsservice-operator-controller-manager-779f4gf9hr   1/1     Running     2 (62m ago)   82m
eda-server-operator-controller-manager-58fdcfc895-px9qz           1/1     Running     2 (62m ago)   82m
resource-operator-controller-manager-fc85f696d-c7t4t              1/1     Running     2 (62m ago)   82m

```

- Access the AAP UI:

Get the AAP access URL, then add the URL-to-IP mapping to the client's hosts file:

```
[admin@microshift ~]$ kubectl get route -n ansible-automation-platform 
NAME   HOST                                               ADMITTED   SERVICE   TLS
aap    aap-ansible-automation-platform.apps.example.com   True       aap   
```

hosts entry:

```
10.210.65.8    aap-ansible-automation-platform.apps.example.com 
```

The admin password can be retrieved with:

```
[admin@microshift ~]$ oc get secret automation-controller-admin-password -n ansible-automation-platform -o jsonpath='{.data.password}' | base64 -d; echo
redhat

[admin@microshift ~]$ oc get secret -n ansible-automation-platform | grep admin-password
aap-admin-password                                           Opaque              1      46m
automation-controller-admin-password                         Opaque              1      116m

[admin@microshift ~]$ kubectl get secret aap-admin-password -n ansible-automation-platform \
  -o jsonpath='{.data.password}' | base64 -d && echo
T9nO5GqsGYUQedlad6EhwwFMtmxaZe3k                <-- Platform Gateway password

```

Note: `redhat` is the Automation Controller password, **not** the Platform Gateway password. Use the Platform Gateway password to log in to the UI.

Finally, access: `https://aap-ansible-automation-platform.apps.example.com` with username `admin` and password `T9nO5GqsGYUQedlad6EhwwFMtmxaZe3k` to open the AAP UI.

![](images/WEBRESOURCE21fe8f35586e4dc434fb4ac80d1ee9b2image.png)

Note:

URL, username, and initial password from a prior environment:

```
https://automation-orchestrator-ansible-automation-platform.apps.example.com
```

Username: admin

Initial password: okai49lttDBcFJyj7u5tZNeFhLznpmp4

- Install Automation Hub and EDA:

After the above steps, logging into the AAP UI reveals only the Automation Controller components. Update the `AnsibleAutomationPlatform` CR to deploy Hub and EDA as well:

```
[admin@microshift ~]$  cat <<EOF | kubectl apply -f -
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatform
metadata:
  name: aap
  namespace: ansible-automation-platform
spec:
  controller:
    name: automation-controller
  hub:
    disabled: false
    storage_type: File
    file_storage_access_mode: ReadWriteOnce
    file_storage_size: 10Gi
  eda:
    disabled: false
  lightspeed:
    disabled: true
EOF

```

Verify CR creation and Pod startup with:

```
[admin@microshift ~]$ kubectl get automationhub -n ansible-automation-platform
NAME      STATUS   MESSAGE
aap-hub   True     All Postgres tasks ran successfully

[admin@microshift ~]$ kubectl get eda -n ansible-automation-platform
NAME      AGE
aap-eda   23m

[admin@microshift ~]$ kubectl get pods -n ansible-automation-platform
NAME                                                              READY   STATUS      RESTARTS      AGE
aap-automationmetricsservice-scheduler-5d88bb4dc9-zflvf           1/1     Running     0             16h
aap-automationmetricsservice-tasks-84b6dd9688-dfbtl               1/1     Running     0             16h
aap-automationmetricsservice-web-657b887f5c-jrn4v                 1/1     Running     0             16h
aap-eda-activation-worker-7787c9f68c-2zsvm                        1/1     Running     0             23m
aap-eda-activation-worker-7787c9f68c-srdln                        1/1     Running     0             23m
aap-eda-api-5579c979d7-n6wp6                                      3/3     Running     0             23m
aap-eda-default-worker-7fb9dbfbdf-jpz4q                           1/1     Running     0             23m
aap-eda-default-worker-7fb9dbfbdf-vwjwr                           1/1     Running     0             23m
aap-eda-event-stream-75f6f489c5-klpth                             2/2     Running     0             23m
aap-gateway-f888fbfc-mkr74                                        2/2     Running     0             27m
aap-gateway-operator-controller-manager-7587f9dc95-8cc4f          1/1     Running     4 (16m ago)   18h
aap-hub-api-6fbccf97c7-7swpt                                      1/1     Running     0             22m
aap-hub-content-86b47f8d5c-7slp9                                  1/1     Running     0             22m
aap-hub-content-86b47f8d5c-jj59j                                  1/1     Running     0             22m
aap-hub-redis-7786c8dc89-ndxhq                                    1/1     Running     0             23m
aap-hub-web-6d76cb74c5-8t5sz                                      1/1     Running     0             23m
aap-hub-worker-77677b944f-5nfc9                                   1/1     Running     0             22m
aap-hub-worker-77677b944f-mzwvs                                   1/1     Running     0             22m
aap-postgres-15-0                                                 1/1     Running     0             16h
aap-redis-0                                                       1/1     Running     0             16h
ansible-lightspeed-operator-controller-manager-5779bb879b-b2znn   1/1     Running     4 (16m ago)   18h
automation-controller-migration-4.8.6-2kjhw                       0/1     Completed   0             18h
automation-controller-operator-controller-manager-7cb4d5d5lwztx   1/1     Running     3 (16m ago)   18h
automation-controller-postgres-15-0                               1/1     Running     0             18h
automation-controller-task-7cbd6b6f86-vl9tw                       4/4     Running     0             22m
automation-controller-web-8cd67f694-fm76m                         3/3     Running     0             22m
automation-hub-operator-controller-manager-7fdd649879-sww5x       1/1     Running     3 (16m ago)   18h
automationmetricsservice-operator-controller-manager-779f4gf9hr   1/1     Running     4 (16m ago)   18h
eda-server-operator-controller-manager-58fdcfc895-px9qz           1/1     Running     4 (16m ago)   18h
resource-operator-controller-manager-fc85f696d-c7t4t              1/1     Running     4 (16m ago)   18h

[admin@microshift ~]$ kubectl get pods -n ansible-automation-platform | grep -E 'hub|eda' | grep -v operator
aap-eda-activation-worker-7787c9f68c-2zsvm                        1/1     Running     0             23m
aap-eda-activation-worker-7787c9f68c-srdln                        1/1     Running     0             23m
aap-eda-api-5579c979d7-n6wp6                                      3/3     Running     0             23m
aap-eda-default-worker-7fb9dbfbdf-jpz4q                           1/1     Running     0             23m
aap-eda-default-worker-7fb9dbfbdf-vwjwr                           1/1     Running     0             23m
aap-eda-event-stream-75f6f489c5-klpth                             2/2     Running     0             23m
aap-hub-api-6fbccf97c7-7swpt                                      1/1     Running     0             22m
aap-hub-content-86b47f8d5c-7slp9                                  1/1     Running     0             23m
aap-hub-content-86b47f8d5c-jj59j                                  1/1     Running     0             23m
aap-hub-redis-7786c8dc89-ndxhq                                    1/1     Running     0             23m
aap-hub-web-6d76cb74c5-8t5sz                                      1/1     Running     0             23m
aap-hub-worker-77677b944f-5nfc9                                   1/1     Running     0             23m
aap-hub-worker-77677b944f-mzwvs                                   1/1     Running     0             23m

```

- Install and deploy the MCP Server:

The MCP Server is managed by `ansible-lightspeed-operator-controller-manager` and must be enabled via the `mcp` field of the `AnsibleAutomationPlatform` CR. It cannot be created manually as a standalone `AnsibleMCPServer` CR.

If the AAP CR was created with the `mcp` field included, skip this step. Otherwise, apply the patch:

```
[admin@microshift ~]$ kubectl patch aap aap -n ansible-automation-platform --type=merge -p '{
  "spec": {
    "mcp": {
      "disabled": false,
      "allow_write_operations": true
    }
  }
}'

```

Confirm that the `AnsibleMCPServer` CR has been created:

```
[admin@microshift ~]$ kubectl get ansiblemcpserver aap-mcp -n ansible-automation-platform
NAME      AGE
aap-mcp   13m

```

Confirm that `ownerReferences` points to `AnsibleAutomationPlatform` (indicating it was created by the Gateway operator and is under controller management):

```
[admin@microshift ~]$ kubectl get ansiblemcpserver aap-mcp -n ansible-automation-platform \
  -o jsonpath='{.metadata.ownerReferences}' | python3 -m json.tool
[
    {
        "apiVersion": "aap.ansible.com/v1alpha1",
        "kind": "AnsibleAutomationPlatform",
        "name": "aap",
        "uid": "f5ec45c5-b505-4375-8c93-0cc8ef3f3a83"
    }
]
```

Monitor deployment progress and confirm status:

MCP Pod status:

```
[admin@microshift ~]$ kubectl get pods -n ansible-automation-platform | grep mcp
aap-mcp-754b899866-sgtmh                                          1/1     Running     0             8m21s
```

Check the MCP CR status:

```
[admin@microshift ~]$ kubectl get ansiblemcpserver aap-mcp -n ansible-automation-platform -o yaml
apiVersion: mcpserver.ansible.com/v1alpha1
kind: AnsibleMCPServer
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: '{"apiVersion":"mcpserver.ansible.com/v1alpha1","kind":"AnsibleMCPServer","metadata":{"name":"aap-mcp","namespace":"ansible-automation-platform"},"spec":{"allow_write_operations":true,"disabled":false,"extra_settings":[{"setting":"IGNORE_CERTIFICATE_ERRORS","value":true}],"no_log":true,"public_base_url":"https://aap-ansible-automation-platform.apps.example.com"}}'
  creationTimestamp: "2026-09-03T08:23:09Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: ansible-mcp-server-operator
    app.kubernetes.io/name: aap-mcp
    app.kubernetes.io/operator-version: ""
    app.kubernetes.io/part-of: aap-mcp
  name: aap-mcp
  namespace: ansible-automation-platform
  ownerReferences:
  - apiVersion: aap.ansible.com/v1alpha1
    kind: AnsibleAutomationPlatform
    name: aap
    uid: f5ec45c5-b505-4375-8c93-0cc8ef3f3a83
  resourceVersion: "277005"
  uid: 0788f7c8-e109-404b-8355-c332965ff984
spec:
  allow_write_operations: true
  extra_settings:
  - setting: IGNORE_CERTIFICATE_ERRORS
    value: true
  image_pull_policy: IfNotPresent
  ingress_type: Route
  no_log: true
  public_base_url: https://aap-ansible-automation-platform.apps.example.com
  service_type: ClusterIP
  set_self_labels: true
status:
  URL: https://aap-mcp-ansible-automation-platform.apps.example.com
  conditions:
  - ansibleResult:
      changed: 2
      completion: "2026-09-03T08:32:05.059449+00:00"
      failures: 0
      ok: 40
      skipped: 13
    lastTransitionTime: "2026-09-03T08:27:42Z"
    message: Awaiting next reconciliation
    reason: Successful
    status: "True"
    type: Running
  - lastTransitionTime: "2026-09-03T08:32:05Z"
    message: Last reconciliation succeeded
    reason: Successful
    status: "True"
    type: Successful
  - lastTransitionTime: "2026-09-03T08:29:13Z"
    message: ""
    reason: ""
    status: "False"
    type: Failure
  image: registry.redhat.io/ansible-automation-platform-27/mcp-server-rhel9@sha256:3751884276ec656608aa3dfb14ec661889d4916a8c7c44a897f2ea04291a5119
  version: 1.0.0

```

Get access information:

```
[admin@microshift ~]$ kubectl get route aap-mcp -n ansible-automation-platform \
  -o jsonpath='{.spec.host}' && echo  
aap-mcp-ansible-automation-platform.apps.example.com

[admin@microshift ~]$ kubectl get ansiblemcpserver aap-mcp -n ansible-automation-platform \
  -o jsonpath='{.status.URL}' && echo
https://aap-mcp-ansible-automation-platform.apps.example.com

```

Access test:

Update the hosts file as follows:

```
10.210.65.8     automation-orchestrator-ansible-automation-platform.apps.example.com  automation-orchestrator-automation-orchestrator.apps.example.com 
10.210.65.8     aap-ansible-automation-platform.apps.example.com aap-mcp-ansible-automation-platform.apps.example.com
```

Access test:

![](images/WEBRESOURCE974b062532896669bacbb8fb45bcc59eimage.png)

**Issues Encountered and Resolutions:**

**Issue 1:**

When deploying AutomationHub, the Hub PVC requests 100 Gi, but the MicroShift node only has approximately 83 Gi of available space. As a result, the created PVC gets stuck and all Hub pods remain in Pending state.

Actual error:

```
oc describe pvc automation-orchestrator-hub-file-storage -n ansible-automation-platform
```

Root cause:

- There is physical disk space available, but not all of it is recognized by the Node / TopoLVM as allocatable space

- AutomationHub's PVC defaults to `file_storage_size: 100Gi`

- This is too large for a single-node MicroShift environment

Resolution:

Find the correct config field:

```
[admin@microshift ~]$ oc get automationhub -A
[admin@microshift ~]$ oc get automationhub automation-orchestrator-hub -n ansible-automation-platform -o yaml
```

Key fields:

```
spec:
  file_storage_access_mode: ReadWriteMany
  file_storage_size: 100Gi
```

Reduce it to 50 Gi:

```
[admin@microshift ~]$ oc get automationhub automation-orchestrator-hub -n ansible-automation-platform -o yaml > /tmp/hub.yml
python3 - <<'PY'
import yaml
p = '/tmp/hub.yml'
with open(p) as f:
    obj = yaml.safe_load(f)
obj['spec']['file_storage_size'] = '50Gi'
with open(p, 'w') as f:
    yaml.safe_dump(obj, f, sort_keys=False)
PY

[admin@microshift ~]$ oc replace -f /tmp/hub.yml
```

Verify again:

```
[admin@microshift ~]$ oc get automationhub automation-orchestrator-hub -n ansible-automation-platform -o jsonpath='{.spec.file_storage_size}'
```

Then delete the old PVC and let it be recreated:

```
[admin@microshift ~]$ oc delete pvc automation-orchestrator-hub-file-storage -n ansible-automation-platform --ignore-not-found
```

**Issue 2:**

PVC Pending — `unsupported access mode`

Root cause:

`topolvm-provisioner` does not support RWX mode.

Confirm the issue:

```
[admin@microshift ~]$ kubectl get pods -n ansible-automation-platform | grep -v Running | grep -v Completed

[admin@microshift ~]$ kubectl describe pod <pod-name> -n ansible-automation-platform | grep -A10 Events

[admin@microshift ~]$ kubectl get pvc -n ansible-automation-platform
```

The Hub file storage PVC is in Pending state, with the following event error:

```
ProvisioningFailed: unsupported access mode: MULTI_NODE_MULTI_WRITER
```

Confirm StorageClass capabilities:

```
[admin@microshift ~]$ kubectl get storageclass
[admin@microshift ~]$ kubectl describe storageclass topolvm-provisioner
```

`topolvm-provisioner` only supports ReadWriteOnce (RWO), not ReadWriteMany (RWX).

Resolution:

Modify the related CR's `file_storage_access_mode` to `ReadWriteOnce`, delete the old PVC, and recreate.

Check the current AutomationHub CR configuration:

```
[admin@microshift ~]$ kubectl get automationhub -n ansible-automation-platform -o yaml | grep -A5 file_storage
```

The output shows `file_storage_access_mode: ReadWriteMany`

Patch the AutomationHub CR to use RWO:

```
[admin@microshift ~]$ kubectl patch automationhub <hub-name> -n ansible-automation-platform \
  --type=merge \
  -p '{"spec":{"file_storage_access_mode":"ReadWriteOnce"}}'
```

Delete the existing Pending PVC and let the operator recreate it:

```
[admin@microshift ~]$ kubectl delete pvc <hub-file-storage-pvc-name> -n ansible-automation-platform
```

Restart the Hub operator (if it is stuck in a reconcile loop):

```
[admin@microshift ~]$ kubectl delete pod -n ansible-automation-platform -l \
  control-plane=automation-hub-operator-controller-manager
```

Wait for the operator to recreate the PVC and start all Hub pods:

```
[admin@microshift ~]$ watch kubectl get pods -n ansible-automation-platform
```

Expected result: all Hub pods become Running, PVC becomes Bound (50 Gi RWO).

**Installation & Deployment (3) — Install Automation Orchestrator (AO):**

AO is a product completely independent of AAP. It provides a drag-and-drop visual IT workflow canvas, built on the Temporal workflow engine, and requires separate installation of its own Operator and CR.

After deployment, AO includes the following components:

| Component | Description |
| -- | -- |
| Backend | AO backend API service |
| UI | AO frontend interface |
| Worker | Task executor |
| Background Worker | Background task processor |
| Temporal | Workflow engine |
| Redis | Cache |
| PostgreSQL | Database (independently deployed) |


AO requires 3 PostgreSQL databases:

- `ao_backend` — AO primary backend database

- `ao_temporal` — Temporal workflow engine database

- `temporal_visibility` — Temporal visibility query database (name is hardcoded and cannot be customized)

- Create a dedicated namespace for AO:

```
[admin@microshift ~]$ kubectl create namespace automation-orchestrator

[admin@microshift ~]$ kubectl create namespace automation-orchestrator-operator-system
```

- Create the OLM OperatorGroup:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: automation-orchestrator-operator-group
  namespace: automation-orchestrator
spec: {}
EOF
```

- Create the OLM Subscription to install the AO Operator, then wait for it to be ready:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: automation-orchestrator-subscription
  namespace: automation-orchestrator
spec:
  channel: stable
  name: automation-orchestrator-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF


```

Verify status with:

```
[admin@microshift ~]$ kubectl get csv -n automation-orchestrator
NAME                                                  DISPLAY                            VERSION             RELEASE   REPLACES   PHASE
automation-orchestrator-operator.v2026.8.1787147047   Automation Orchestrator Operator   2026.8.1787147047                        Succeeded

[admin@microshift ~]$ kubectl get pods -n automation-orchestrator
NAME                                                              READY   STATUS    RESTARTS   AGE
automation-orchestrator-operator-controller-manager-5b7bc6chmnh   1/1     Running   0          16m

[admin@microshift ~]$ kubectl get pods -n automation-orchestrator
automation-orchestrator                  automation-orchestrator-operator-system 
```

- Deploy PostgreSQL:

Create PostgreSQL credentials Secret:

```
[admin@microshift ~]$ kubectl create secret generic ao-postgres-credentials \
  -n automation-orchestrator \
  --from-literal=POSTGRESQL_USER=aouser \
  --from-literal=POSTGRESQL_PASSWORD=aopassword123 \
  --from-literal=POSTGRESQL_ADMIN_PASSWORD=aoadminpassword123 \
  --from-literal=POSTGRESQL_DATABASE=ao_backend
```

Create PVC:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ao-postgres-pvc
  namespace: automation-orchestrator
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
EOF
```

- Create the PostgreSQL StatefulSet:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ao-postgres
  namespace: automation-orchestrator
spec:
  serviceName: ao-postgres
  replicas: 1
  selector:
    matchLabels:
      app: ao-postgres
  template:
    metadata:
      labels:
        app: ao-postgres
    spec:
      containers:
      - name: postgres
        image: registry.redhat.io/rhel9/postgresql-15:1
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRESQL_USER
          valueFrom:
            secretKeyRef:
              name: ao-postgres-credentials
              key: POSTGRESQL_USER
        - name: POSTGRESQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ao-postgres-credentials
              key: POSTGRESQL_PASSWORD
        - name: POSTGRESQL_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ao-postgres-credentials
              key: POSTGRESQL_ADMIN_PASSWORD
        - name: POSTGRESQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: ao-postgres-credentials
              key: POSTGRESQL_DATABASE
        volumeMounts:
        - name: data
          mountPath: /var/lib/pgsql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: ao-postgres-pvc
EOF
```

- Create PostgreSQL Services (Headless + ClusterIP), then wait for readiness:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ao-postgres
  namespace: automation-orchestrator
spec:
  selector:
    app: ao-postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: ao-postgres-svc
  namespace: automation-orchestrator
spec:
  selector:
    app: ao-postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF
```

Verify and wait for the PostgreSQL Pod to be ready:

```
[admin@microshift ~]$ kubectl get pods -n automation-orchestrator
NAME                                                              READY   STATUS    RESTARTS   AGE
ao-postgres-0                                                     1/1     Running   0          75s
automation-orchestrator-operator-controller-manager-5b7bc6chmnh   1/1     Running   0          55m

```

- Initialize databases:

Note: PostgreSQL 15 changed the permission policy for the `public` schema. Non-owner users must be explicitly granted access.

Create the required databases:

```
[admin@microshift ~]$ kubectl exec -n automation-orchestrator ao-postgres-0 -- bash -c "
  psql -U postgres -c 'CREATE DATABASE ao_backend;'
  psql -U postgres -c 'CREATE DATABASE ao_temporal;'
  psql -U postgres -c 'CREATE DATABASE temporal_visibility;'
"
```

**Note:** `temporal_visibility` is hardcoded by the Temporal engine. You must use this exact name — it cannot be customized.

Grant `aouser` access to all databases:

```
[admin@microshift ~]$ kubectl exec -n automation-orchestrator ao-postgres-0 -- bash -c "
  # ao_backend
  psql -U postgres -c 'GRANT ALL PRIVILEGES ON DATABASE ao_backend TO aouser;'
  psql -U postgres -d ao_backend -c 'GRANT ALL ON SCHEMA public TO aouser;'
  psql -U postgres -d ao_backend -c 'ALTER DATABASE ao_backend OWNER TO aouser;'

  # ao_temporal
  psql -U postgres -c 'GRANT ALL PRIVILEGES ON DATABASE ao_temporal TO aouser;'
  psql -U postgres -d ao_temporal -c 'GRANT ALL ON SCHEMA public TO aouser;'
  psql -U postgres -d ao_temporal -c 'ALTER DATABASE ao_temporal OWNER TO aouser;'

  # temporal_visibility
  psql -U postgres -c 'GRANT ALL PRIVILEGES ON DATABASE temporal_visibility TO aouser;'
  psql -U postgres -d temporal_visibility -c 'GRANT ALL ON SCHEMA public TO aouser;'
  psql -U postgres -d temporal_visibility -c 'ALTER DATABASE temporal_visibility OWNER TO aouser;'
"
```

- Create AO database connection Secrets:

Backend database Secret:

```
[admin@microshift ~]$ kubectl create secret generic backend-db-secret \
  -n automation-orchestrator \
  --from-literal=database=ao_backend \
  --from-literal=username=aouser \
  --from-literal=password=aopassword123
```

Temporal database Secret:

```
[admin@microshift ~]$ kubectl create secret generic temporal-db-secret \
  -n automation-orchestrator \
  --from-literal=database=ao_temporal \
  --from-literal=username=aouser \
  --from-literal=password=aopassword123
```

- Create the AutomationOrchestrator CR:

```
[admin@microshift ~]$ cat <<EOF | kubectl apply -f -
apiVersion: aap.ansible.com/v1alpha1
kind: AutomationOrchestrator
metadata:
  name: automation-orchestrator
  namespace: automation-orchestrator
spec:
  ingress:
    type: Route
  postgres:
    host: ao-postgres.automation-orchestrator.svc.cluster.local
    port: 5432
    sslMode: disable
    backendDatabase:
      secretRef:
        name: backend-db-secret
    temporalDatabase:
      secretRef:
        name: temporal-db-secret
EOF
```

- Monitor deployment progress:

```
[admin@microshift ~]$ watch kubectl get pods -n automation-orchestrator

[admin@microshift ~]$ kubectl get automationorchestrator -n automation-orchestrator -w

[admin@microshift ~]$ kubectl logs -n automation-orchestrator-operator-system -l control-plane=controller-manager -f
```

- Get access information:

Get the AO URL:

```
[admin@microshift ~]$ kubectl get route automation-orchestrator -n automation-orchestrator -o jsonpath='{.spec.host}'
```

Example output:

```
automation-orchestrator-automation-orchestrator.apps.example.com
automation-orchestrator-automation-orchestrator.apps.example.com
```

Get the initial admin password:

```
[admin@microshift ~]$ kubectl get secret automation-orchestrator-initial-admin-password \
  -n automation-orchestrator \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Example output:

```
[admin@microshift ~]$ kubectl get secret automation-orchestrator-initial-admin-password 
-n automation-orchestrator 
-o jsonpath='{.data.password}' | base64 -d && echo
d3HevqYRi8W80PD8VF8xpn4qdJMnQgyq
```

Add the hostname resolution to the local hosts file:

```
10.210.65.8     automation-orchestrator-ansible-automation-platform.apps.example.com  automation-orchestrator-automation-orchestrator.apps.example.com
```

- Final verification:

```
[admin@microshift ~]$ kubectl get pods -n automation-orchestrator

[admin@microshift ~]$ kubectl get automationorchestrator automation-orchestrator -n automation-orchestrator -o wide
```

- Browser access:

![](images/WEBRESOURCE7309a784b9ba16daa3ef0dc38cfe2695image.png)
