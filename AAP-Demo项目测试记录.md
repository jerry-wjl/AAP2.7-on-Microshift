**项目地址：**
[https://github.com/RedhatOfficial/aap-demo](https://github.com/RedhatOfficial/aap-demo)
<br>


**项目说明：**
Deploy AAP to a local MicroShift cluster in minutes, is a LOCAL DEVELOPMENT tool and must NEVER be used in production.
<br>


**前置条件：**

- CRC (OpenShift Local) — 

- 16 GB RAM minimum — default VM allocation is 16 GB (override with CRC_MEMORY=24576 aap-demo create for 24 GB)

- 16 CPU — 测试过程中发现当CPU数量为8时，在部分场景中出现因无法分配CPU而导致pod无法启动的现象

- 60 GB + disk space - for aap-operator deploy

- Pull secret — download from console.redhat.com

- 如果使用虚拟机部署，确保虚拟机启动了硬件虚拟化

- 需要提前部署以下安装包：libvirt-daemon，libvirt-daemon-driver-storage，libvirt-daemon-driver-network，qemu-kvm

- 部署必须使用普通用户（创建普通用户admin，并在sudoers中提权）

- 系统必须以FQDN格式命名，并更新/etc/hosts
<br>

**具体操作步骤：**

1. 访问：[https://console.redhat.com/openshift/create/local](https://console.redhat.com/openshift/create/local)
先下载openshift-local和pull-secret：
![](images/WEBRESOURCEd6a8349c6b4c5f294521b1c8f3378890image.png)

pull-secret.txt存放位置：
```
[root@aap-demo ~]# mkdir -p ~/.aap-demo

[root@aap-demo ~]# cd .aap-demo/
[root@aap-demo .aap-demo]# cat pull-secret.txt 
{"auths":{"cloud.openshift.com":{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}}}
```
<br>

2. 下载和部署crc（openshift local）安装工具：
```
[admin@aap-demo ~]$ git clone https://github.com/RedHatOfficial/aap-demo.git

[admin@aap-demo ~]$ cd aap-demo && ./install.sh

[admin@aap-demo ~]$ sudo cp crc-linux-*/crc /usr/local/bin/
```
<br>

3. 安装和启动crc：
```
[admin@aap-demo ~]$ crc setup
INFO Using bundle path /home/admin/.crc/cache/crc_libvirt_4.22.7_amd64.crcbundle 
INFO Checking if running as non-root              
INFO Checking if running inside WSL2              
INFO Checking if crc-admin-helper executable is cached 
INFO Checking if running on a supported CPU architecture 
INFO Checking if crc executable symlink exists    
INFO Checking minimum RAM requirements            
INFO Checking if Virtualization is enabled        
INFO Checking if KVM is enabled                   
INFO Checking if libvirt is installed             
INFO Checking if user is part of libvirt group    
INFO Adding user to libvirt group                 
INFO Using root access: Adding user to the libvirt group 
INFO Checking if active user/process is currently part of the libvirt group 
INFO Checking if libvirt daemon is running        
INFO Checking if a supported libvirt version is installed 
INFO Checking if crc-driver-libvirt is installed  
INFO Installing crc-driver-libvirt                
INFO Checking crc daemon systemd service          
INFO Setting up crc daemon systemd service        
INFO Checking crc daemon systemd socket units     
INFO Setting up crc daemon systemd socket units   
INFO Checking if vsock is correctly configured    
INFO Setting up vsock support                     
INFO Using root access: Setting CAP_NET_BIND_SERVICE capability for /usr/local/bin/crc executable 
INFO Using root access: Creating udev rule for /dev/vsock 
INFO Using root access: Changing permissions for /etc/udev/rules.d/99-crc-vsock.rules to 644  
INFO Using root access: Reloading udev rules database 
INFO Using root access: Loading vhost_vsock kernel module 
INFO Using root access: Creating file /etc/modules-load.d/vhost_vsock.conf 
INFO Using root access: Changing permissions for /etc/modules-load.d/vhost_vsock.conf to 644  
INFO Checking if CRC bundle is extracted in '$HOME/.crc' 
INFO Checking if /home/admin/.crc/cache/crc_libvirt_4.22.7_amd64.crcbundle exists 
INFO Getting bundle for the CRC executable        
INFO Downloading bundle: /home/admin/.crc/cache/crc_libvirt_4.22.7_amd64.crcbundle... 
5.81 GiB / 5.81 GiB [--------------------------------------------------------------------------------------------------------------------------------] 100.00% 180.59 MiB/s
INFO Uncompressing /home/admin/.crc/cache/crc_libvirt_4.22.7_amd64.crcbundle 
crc.qcow2:  21.06 GiB / 21.06 GiB [-------------------------------------------------------------------------------------------------------------------------------] 100.00%
oc:  129.79 MiB / 129.79 MiB [------------------------------------------------------------------------------------------------------------------------------------] 100.00%
Your system is correctly setup for using CRC. Use 'crc start' to start the instance
```

```
[admin@aap-demo ~]$ crc start
INFO Using bundle path /home/admin/.crc/cache/crc_libvirt_4.22.7_amd64.crcbundle 
INFO Checking if running as non-root              
INFO Checking if running inside WSL2              
INFO Checking if crc-admin-helper executable is cached 
INFO Checking if running on a supported CPU architecture 
INFO Checking if crc executable symlink exists    
INFO Checking minimum RAM requirements            
INFO Checking if Virtualization is enabled        
INFO Checking if KVM is enabled                   
INFO Checking if libvirt is installed             
INFO Checking if user is part of libvirt group    
INFO Checking if active user/process is currently part of the libvirt group 
INFO Checking if libvirt daemon is running        
INFO Checking if a supported libvirt version is installed 
INFO Checking if crc-driver-libvirt is installed  
INFO Checking crc daemon systemd socket units     
INFO Checking if vsock is correctly configured    
INFO Loading bundle: crc_libvirt_4.22.7_amd64...  
CRC requires a pull secret to download content from Red Hat.
You can copy it from the Pull Secret section of https://console.redhat.com/openshift/create/local.
? Please enter the pull secret             <--此处需要手动粘贴pull-secret的内容

WARN Cannot add pull secret to keyring: The name is not activatable 
INFO Creating CRC VM for OpenShift 4.22.7...      
INFO Generating new SSH key pair...               
INFO Generating new password for the kubeadmin user 
INFO Starting CRC VM for openshift 4.22.7...      
INFO CRC instance is running with IP 127.0.0.1    
INFO CRC VM is running                            
INFO Updating authorized keys...                  
INFO Configuring shared directories               
INFO Check internal and public DNS query...       
INFO Check DNS query from host...                 
INFO Verifying validity of the kubelet certificates... 
INFO Starting kubelet service                     
INFO Waiting for kube-apiserver availability... [takes around 2min] 
INFO Adding user's pull secret to the cluster...  
INFO Updating SSH key to machine config resource... 
INFO Overriding password for developer user       
INFO Changing the password for the users          
INFO Updating cluster ID...                       
INFO Updating root CA cert to admin-kubeconfig-client-ca configmap... 
INFO Starting openshift instance... [waiting for the cluster to stabilize] 
INFO 3 operators are progressing: openshift-controller-manager, operator-lifecycle-manager-packageserver, service-ca 
INFO 3 operators are progressing: authentication, image-registry, openshift-controller-manager 
INFO Operator authentication is progressing       
INFO All operators are available. Ensuring stability... 
INFO Operators are stable (2/3)...                
INFO Operators are stable (3/3)...                
INFO Waiting until the user's pull secret is written to the instance disk... 
INFO Adding crc-admin and crc-developer contexts to kubeconfig... 
Started the OpenShift cluster.


The server is accessible via web console at:
  https://console-openshift-console.apps-crc.testing

Log in as administrator:
  Username: kubeadmin
  Password: BSds6-nSzi2-mbB4W-fQISw

Log in as user:
  Username: developer
  Password: developer

Use the 'oc' command line interface:
  $ eval $(crc oc-env)
  $ oc login -u developer https://api.crc.testing:6443

```
<br>

4，准备NFS Storage Class：

由于AAP的automation hub默认使用基于NFS的Storage Class，所以在执行aap-demo deploy之前要先执行NFS相关的配置，否则aap-demo deploy到Automation Hub的相关Pod会报错。

如果crc已经启动，则建议重启虚拟机，重启后crc保持关闭状态，此时可以进行NFS相关配置；如果不重启虚拟机，则crc也要手动被关闭。

假设crc已经关闭，则通过以下命令修改crc核心参数，防止部署过程中资源不足：
```
[admin@aap-demo ~]$ crc config set disk-size 60
[admin@aap-demo ~]$ crc config set memory 24576
[admin@aap-demo ~]$ crc config set cpus 8
```
<br>

启动crc：
```
[admin@aap-demo ~]$ crc start
```
<br>

导入凭据模板（可选）：
```
[admin@aap-demo ~]$ export KUBECONFIG=~/.crc/machines/crc/kubeconfig
```
<br>

备注：
如果需要清理之前已经部署的CRC并重新配置，则执行以下步骤：
```
[admin@aap-demo ~]$ crc stop
[admin@aap-demo ~]$ crc cleanup

[admin@aap-demo ~]$ crc config set disk-size 60
[admin@aap-demo ~]$ crc config set memory 24576
[admin@aap-demo ~]$ crc config set cpus 8

[admin@aap-demo ~]$ crc setup
[admin@aap-demo ~]$ crc start

[admin@aap-demo ~]$ export KUBECONFIG=~/.crc/machines/crc/kubeconfig
```
<br>

配置CRC内置的nfs服务：
```
[admin@aap-demo ~]$ ssh -i ~/.crc/machines/crc/id_ed25519 core@127.0.0.1 -p 2222 
[admin@aap-demo ~]$ sudo mkdir -p /var/srv/nfs/aap-hub 
[admin@aap-demo ~]$ sudo chmod 777 /var/srv/nfs/aap-hub 
[admin@aap-demo ~]$ echo '/var/srv/nfs/aap-hub *(rw,sync,no_subtree_check,no_root_squash)' | sudo tee /etc/exports 
[admin@aap-demo ~]$ sudo exportfs -a 
[admin@aap-demo ~]$ sudo systemctl enable --now nfs-server
```
<br>

创建命名空间并赋予 privileged 特权放行：
```
[admin@aap-demo ~]$ kubectl create ns nfs-provisioner
[admin@aap-demo ~]$ kubectl label ns nfs-provisioner pod-security.kubernetes.io/enforce=privileged --overwrite
[admin@aap-demo ~]$ kubectl patch scc privileged --type='json' -p='[{"op": "add", "path": "/users/-", "value": "system:serviceaccount:nfs-provisioner:nfs-client-provisioner"}]'
```
<br>

部署包含完整 RBAC 权限的 NFS Provisioner 和 nfs-local-rwx 存储类：
```
[admin@aap-demo ~]$ kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          securityContext:
            privileged: true
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: "127.0.0.1"
            - name: NFS_PATH
              value: "/var/srv/nfs/aap-hub"
      volumes:
        - name: nfs-client-root
          nfs:
            server: "127.0.0.1"
            path: "/var/srv/nfs/aap-hub"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes", "persistentvolumes", "persistentvolumeclaims", "storageclasses"]
    verbs: ["get", "list", "watch", "create", "delete", "update"]
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "update", "patch", "get", "list", "watch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-local-rwx
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```
<br>

清除内置存储类的 Default 标记，并把 NFS 设为唯一默认存储类：

移除内置存储类的默认标记：
```
[admin@aap-demo ~]$ kubectl patch storageclass crc-csi-hostpath-provisioner -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```
<br>

将 nfs-local-rwx 设为全局唯一的默认存储类：
```
[admin@aap-demo ~]$ kubectl patch storageclass nfs-local-rwx -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
<br>

移除 Node 潜在的污点，确保无阻碍调度：
```
[admin@aap-demo ~]$ kubectl uncordon crc
```

确保pod处于running状态：
```
[admin@aap-demo ~]$ kubectl get pods -n nfs-provisioner
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-649769b78d-zqb76   1/1     Running   0          9h
```

确保storage class状态如下：
```
[admin@aap-demo ~]$ kubectl get sc
NAME                           PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
crc-csi-hostpath-provisioner   kubevirt.io.hostpath-provisioner              Retain          WaitForFirstConsumer   false                  25d
nfs-local-rwx (default)        k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate              true                   9h
```

4. 安装aap-demo：
环境安装operator-sdk：
```
[admin@aap-demo ~]$ export CURL_CA_BUNDLE=""
[admin@aap-demo ~]$ echo 'insecure' >> ~/.curlrc
```
<br>

执行aap-demo部署：
```
[admin@aap-demo ~]$ aap-demo deploy
```
<br>

部署后的状态使用以下命令获取：
```
[admin@aap-demo ~]$ aap-demo status

AAP Demo Status
===============
Tool:        1.0.3 (856aad9)
Built:       2026-08-21T14:19:25-05:00

Infra:       OpenShift Local (CRC)
Cluster:     running (crc-microshift)

TLS:
----
  Ingress CA file:   /home/admin/.aap-demo/crc-ingress-ca.crt
  System trust:      trusted
  Browser trust:     unknown (install nss-tools for Chrome/Firefox)

Kubeconfig:  /home/admin/.crc/machines/crc/kubeconfig
Source:      /home/admin/aap-demo (branch: main)
Repo:        https://github.com/RedHatOfficial/aap-demo.git

VM:
---
  OS:           Red Hat Enterprise Linux release 9.8 (Plow)
  OpenShift:    
  CPUs:         8
  Memory:       15Gi / 23Gi (8.0Gi available)
  Load:         2.49 2.63 1.99
  Disk:         39G/100G (39% used)

Namespaces:
-----------
  aap-operator                   26/26 pods   aap
  automation-orchestrator        1/1 pods
  hostpath-provisioner           1/1 pods
  nfs-provisioner                1/1 pods

AAP Deployments:
----------------
  https://aap-aap-operator.apps-crc.testing
  https://aap-mcp-aap-operator.apps.127.0.0.1.nip.io

Credentials:
------------
  aap-operator:        admin / MnNWItHbbLC6862JG5gNT7LgAujuOsPf


Addons:
-------
  mcp-server      enabled
  portal          disabled
  setup-pah       disabled
  ao              enabled
  apme-eap        disabled
  local-cache     disabled
  product-demos   disabled
  product-demo-satellite disabled

```
<br>

注意：
在部署的过程中可能碰到的情况是实际上部署已经完成，但部署任务没有成功提示，且始终占用终端。
这种情况可先通过以下命令查看部署aap-operator执行情况，确保anible任务执行完成。
```
[admin@aap-demo ~]$ kubectl logs -n aap-operator deployment/resource-operator-controller-manager -c manager --tail=50 -f
```
<br>

或者可以执行aap-demo status确认状态，并确保所有的pod都已经处于运行状态。
```
......
Namespaces:
-----------
  aap-operator                   26/26 pods   aap
  automation-orchestrator        1/1 pods
  hostpath-provisioner           1/1 pods
  nfs-provisioner                1/1 pods
......
```
<br>

此时可以执行以下命令，强行触发 Operator 刷状态（如果发现所有 Pod 都是健康的，只是 Condition 没更新，直接给主 CR 打个无害的 annotation，迫使 Operator 重新检查并把状态刷成 Successful）。
```
[admin@aap-demo ~]$ kubectl annotate ansibleautomationplatform --all -n aap-operator force-reconcile=$(date +%s) --overwrite
```
<br>

5. 访问aap-demo环境：
访问aap-demo环境的链接通过aap-demo status命令获得：
```
AAP Deployments:
----------------
  https://aap-aap-operator.apps-crc.testing                    <-- AAP访问链接
  https://aap-mcp-aap-operator.apps.127.0.0.1.nip.io
```
<br>

注意访问必须通过域名，如果通过建立隧道穿透跳板机访问则需要在终端修改hosts文件，例如在本人windows主机的hosts文件中修改如下：
```
127.0.0.1  aap-aap-operator.apps-crc.testing
```
<br>

从终端建立隧道：
```
C:\Users\jerrywjl>ssh -L 10443:192.168.72.90:443 lab-user@bastion-2vvct.cluster-2vvct.dyn.redhatworkshops.io
```
<br>

此时访问链接为：
```
https://aap-aap-operator.apps-crc.testing:10443
```
<br>

首次登录默认用户为admin，输入的admin密码可通过aap-demo status获得：
```
......
Credentials:
------------
  aap-operator:        admin / MnNWItHbbLC6862JG5gNT7LgAujuOsPf
......
```
<br>

或者通过以下命令获得：
```
[admin@aap-demo ~]$ kubectl get secret -n aap-operator -o json | jq -r '.items[] | select(.metadata.name | contains("admin-password")) | "\(.metadata.name): " + (.data.password // .data.admin_password | @base64d)'
aap-admin-password: MnNWItHbbLC6862JG5gNT7LgAujuOsPf
aap-controller-admin-password: yfDXshRSMaknCGRNwBfeJBU5TonAS4jf
aap-eda-admin-password: 7WvWVTzcc3aXLjaSPFHaGamyPb5vTgM4
aap-hub-admin-password: yXCOZqyKpjGDROcKcmpdcFuDTMocS1tT
```
<br>

5. 启用aap-demo附加组件（add-ons）：
aap-demo有以下附件组件，可通过命令enable/disable来启用或关闭。
```
aap-demo enable              # List all addons
aap-demo enable portal       # Installs Automation Portal
aap-demo enable setup-pah    # Configures Private Automation Hub Credentials
aap-demo enable mcp-server   # MCP server for AI assistants
aap-demo enable ao           # Automation Orchestrator (GA; no aapctl required — see addons/ao/README.md)
aap-demo enable apme-eap     # Early Access Program only for APME
aap-demo enable local-cache  # Caches AAP containers locally so you don't re-download after destroy/create

# Ansible Product Demos - Official demo content from ansible/product-demos
aap-demo enable product-demos                # Five domains at once (includes base; Satellite opt-in)
aap-demo enable product-demo-satellite       # Satellite demos (requires a Satellite server)
aap-demo disable addon_name                  # Disables addon
```
<br>

但当执行enable ao时（automation orchestration）会出现以下报错：
```
[admin@aap-demo ~]$ aap-demo enable ao
Enabling addon: ao
  Source: /home/admin/aap-demo/addons/ao/deploy.sh (1.0.3 (856aad9))
Creating namespace and SCC grants...
namespace/automation-orchestrator created
✓ Namespace ready
Checking AAP redhat-operators catalog (for index image and pull secret)...
Creating AO CatalogSource in automation-orchestrator...
  Index: redhat/redhat-operator-index:v4.22
  Ensuring MicroShift 4.22+ signature policy allows registry.redhat.io...
  Container signature policy already relaxed
secret/redhat-operators-pull-secret created
catalogsource.operators.coreos.com/redhat-operators created
  Waiting for CatalogSource READY...
✓ CatalogSource READY
  Waiting for operator index to sync...

✓ automation-orchestrator-operator found in catalog
✓ Operator package found in AO catalog (automation-orchestrator, stable)
✓ Ingress host: automation-orchestrator.apps-crc.testing
✓ CloudNativePG operator ready
Creating PostgreSQL cluster for Automation Orchestrator...
secret/orchestrator-postgres-secret created
secret/temporal-postgres-secret created
secret/temporal-visibility-postgres-secret created
database.postgresql.cnpg.io/orchestrator created
database.postgresql.cnpg.io/temporal created
database.postgresql.cnpg.io/temporal-visibility created
Error from server (InternalError): error when creating "STDIN": Internal error occurred: failed calling webhook "mcluster.cnpg.io": failed to call webhook: Post "https://cnpg-webhook-service.cnpg-system.svc:443/mutate-postgresql-cnpg-io-v1-cluster?timeout=10s": no endpoints available for service "cnpg-webhook-service"
ERROR: Failed to apply PostgreSQL manifests.
```

导致该问题的原因：
在 OpenShift/CRC 环境中启用 Automation Orchestrator (ao) 时，核心障碍在于 CloudNativePG (CNPG) 数据库算子的安装与权限缺失。
<br>
具体原因包括以下几个：
- Webhook 挂钩超时卡死：
	- 现象：执行 aap-demo enable ao 时，报 failed calling webhook "mcluster.cnpg.io": no endpoints available 错误。
	- 原因：集群中残留了 CNPG 的 Webhook 校验配置，但 cnpg-system 命名空间下实际并没有运行任何 CNPG Operator Pod。

- 数据库 Cluster 无法创建：
	- 现象：清理 Webhook 后再次执行，卡在 PostgreSQL cluster not ready after 10 minutes。
	- 原因：CNPG 算子未真正部署运行，导致提交给 Kubernetes 的 orchestrator-postgres 数据库资源无人处理，完全没有生成底层数据库 Pod。

- CNPG Deployment 部署超时：
	- 现象：尝试手动部署 CNPG 后，cnpg-controller-manager 报 exceeded its progress deadline。
	- 原因：OpenShift 的安全上下文约束（SCC）拦截了 CNPG ServiceAccount，且当前终端未安装 oc 命令，导致常规的 oc adm 授权失败。

解决方法：
清理残留的 CNPG Webhook 约束：
```
[admin@aap-demo ~]$ kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/name=cloudnative-pg --ignore-not-found
[admin@aap-demo ~]$ kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/name=cloudnative-pg --ignore-not-found
```

安装 CNPG Operator 并授予 OpenShift Privileged 权限：
创建命名空间并标记放行：
```
[admin@aap-demo ~]$ kubectl create ns cnpg-system --dry-run=client -o yaml | kubectl apply -f -
[admin@aap-demo ~]$ kubectl label ns cnpg-system pod-security.kubernetes.io/enforce=privileged --overwrite
```
<br>

部署 CNPG 算子 (v1.22.1)：
```
[admin@aap-demo ~]$ kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.1.yaml
```
<br>

用 kubectl 纯命令直接修补 OpenShift SCC 特权：
```
[admin@aap-demo ~]$ kubectl patch scc privileged --type='json' -p='[{"op": "add", "path": "/users/-", "value": "system:serviceaccount:cnpg-system:cnpg-manager"}]' --ignore-not-found
```
<br>

重启并等待 CNPG Operator 进入 Ready 状态：
```
[admin@aap-demo ~]$ kubectl rollout restart deployment cnpg-controller-manager -n cnpg-system
[admin@aap-demo ~]$ kubectl rollout status deployment/cnpg-controller-manager -n cnpg-system --timeout=120s
```
<br>

完成后重启add-on：
```
[admin@aap-demo ~]$ aap-demo enable ao
```
<br>

根据部署后的信息访问AO的Portal：
```
........
✓ Automation Orchestrator operator and instance applied

  URL:      https://automation-orchestrator.apps-crc.testing
  Username: admin
  Password: ykAi5xd03lj3lTMo8TAlmpcOEsr/hyN/
  Status:   kubectl get pods -n automation-orchestrator
  Saved to config: ADDONS=mcp-server,ao
```
<br>

通过建立隧道访问AO：

先更新终端的hosts文件如下：
```
127.0.0.1  aap-aap-operator.apps-crc.testing  automation-orchestrator.apps-crc.testing
```
<br>

建立隧道：
```
C:\Users\jerrywjl>ssh -L 20443:192.168.72.90:443 lab-user@bastion-2vvct.cluster-2vvct.dyn.redhatworkshops.io
```
<br>

通过以下链接访问AO的portal：
```
https://automation-orchestrator.apps-crc.testing:20443
```
![](images/WEBRESOURCE359f5012fae871b8b48620fe12929482image.png)
<br>

启用portal：
```
[admin@aap-demo ~]$ curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Downloading https://get.helm.sh/helm-v3.21.4-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm

[admin@aap-demo ~]$ helm version --short
v3.21.4+g813176c

[admin@aap-demo ~]$ aap-demo enable portal
Enabling addon: portal
  Source: /home/admin/aap-demo/addons/portal/deploy.sh (1.0.3 (856aad9))
Checking prerequisites...
✓ Cluster architecture: amd64
✓ Using x86 profile (Red Hat RHDH images)
✓ Prerequisites met
Setting up portal namespace: redhat-rhaap-portal
namespace/redhat-rhaap-portal created
namespace/redhat-rhaap-portal labeled
clusterrolebinding.rbac.authorization.k8s.io/system:openshift:scc:anyuid:redhat-rhaap-portal created
clusterrolebinding.rbac.authorization.k8s.io/system:openshift:scc:privileged:redhat-rhaap-portal created
secret/redhat-operators-pull-secret created
serviceaccount/default patched
✓ Portal namespace ready
Fetching AAP credentials...
✓ AAP accessible at: aap-aap-operator.apps-crc.testing
Selecting AAP organization...
✓ Using organization: Default (ID: 1)
Creating OAuth application in AAP...
✓ OAuth app ready (ID: 1)
Enabling OAuth token creation for external users...
✓ OAuth tokens enabled
Generating AAP API token...
✓ API token generated
Configuring registry credentials...
✓ Using existing registry.redhat.io credentials from cluster
Creating registry secret in OpenShift...
secret/redhat-rhaap-portal-dynamic-plugins-registry-auth created
✓ Registry secret created
Getting cluster information...
✓ Cluster base URL: apps-crc.testing
Creating AAP credentials secret...
secret/secrets-rhaap-portal created
✓ AAP credentials secret created
✓ AAP host URL: https://aap-aap-operator.apps-crc.testing
Creating Helm values file...
✓ Helm values created (x86 profile)
Installing Helm chart...
Adding OpenShift Helm Charts repository...
"openshift-helm-charts" has been added to your repositories
Installing Helm release...
NAME: redhat-rhaap-portal
LAST DEPLOYED: Sun Aug 23 23:27:57 2026
NAMESPACE: redhat-rhaap-portal
STATUS: deployed
REVISION: 1
TEST SUITE: None
✓ Helm chart installed
Waiting for portal deployment to be ready...
Waiting for deployment "redhat-rhaap-portal" rollout to finish: 0 out of 1 new replicas have been updated...
Waiting for deployment "redhat-rhaap-portal" rollout to finish: 0 of 1 updated replicas are available...
⚠️  Deployment taking longer than expected
Check status with: kubectl get pods -n redhat-rhaap-portal
Proceeding anyway...
Updating OAuth redirect URI...
✓ OAuth redirect URI updated: https://redhat-rhaap-portal-redhat-rhaap-portal.apps-crc.testing/api/auth/rhaap/handler/frame
Verifying AAP host URL in portal pod...
⚠️  Could not read AAP_HOST_URL from portal pod
⚠️  AAP host URL verification failed (portal may still work)
Verifying OAuth client credentials...
❌ Portal pod cannot reach AAP token endpoint at https://aap-aap-operator.apps-crc.testing/o/token/
   On CRC/MicroShift, nip.io resolves to 127.0.0.1 inside pods.
   Re-run: aap-demo enable portal
⚠️  OAuth client verification failed (portal may still work)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Portal addon enabled successfully!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Portal URL: https://redhat-rhaap-portal-redhat-rhaap-portal.apps-crc.testing
Profile: x86 (Red Hat RHDH hub image from chart)

Next steps:
1. Open the portal URL in your browser
2. Click 'Sign In'
3. Authenticate with AAP credentials (admin / <aap-admin-password>)
4. Browse AAP job templates in the catalog

Check status: aap-demo status portal
Disable: aap-demo disable portal

  Saved to config: ADDONS=mcp-server,ao,portal
 
```
<br>

虽然portal已经enable了，但是其中有一个错误：
```
Verifying OAuth client credentials...
❌ Portal pod cannot reach AAP token endpoint at https://aap-aap-operator.apps-crc.testing/o/token/
   On CRC/MicroShift, nip.io resolves to 127.0.0.1 inside pods.
   Re-run: aap-demo enable portal
⚠️  OAuth client verification failed (portal may still work)
```
<br>

主要原因有两个：
- CRC/MicroShift 内部 DNS 解析死锁：Portal Pod 在容器内尝试连接 AAP 的 OAuth 端点（[
- CRC 节点 CPU 资源争抢（Pod Pending）：Portal 默认请求的 CPU 配额较高，在单节点 CRC 环境中容易触发 1 Insufficient cpu，导致新 Pod 卡在 Pending 无法被调度。

针对CRC环境使用以下命令进行修复：
1. 降低 CPU 请求配额，解决 Pending 调度问题
```
[admin@aap-demo ~]$ kubectl patch deployment redhat-rhaap-portal -n redhat-rhaap-portal --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/cpu", "value": "100m"},
  {"op": "replace", "path": "/spec/template/spec/initContainers/0/resources/requests/cpu", "value": "100m"}
]'
```
<br>

2. 注入 hostAliases，将 AAP 域名强制映射到集群内网 Ingress Router 的 ClusterIP (10.217.4.100)
```
[admin@aap-demo ~]$ kubectl patch deployment redhat-rhaap-portal -n redhat-rhaap-portal --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/hostAliases", "value": [
    {"ip": "10.217.4.100", "hostnames": ["aap-aap-operator.apps-crc.testing"]}
  ]}
]'
```
<br>

3. 等待部署滚动更新完成
```
[admin@aap-demo ~]$ kubectl rollout status deployment/redhat-rhaap-portal -n redhat-rhaap-portal --timeout=120s
```
<br>

4. 确认 Pod 处于 2/2 Running 状态：
```
[admin@aap-demo ~]$ kubectl get pods -n redhat-rhaap-portal 
NAME                                   READY   STATUS    RESTARTS   AGE
redhat-rhaap-portal-7cd8cf9464-n9dp7   2/2     Running   0          7m33s
redhat-rhaap-portal-postgresql-0       1/1     Running   0          8m28s
```
<br>

5. 验证容器内部能否直达 AAP OAuth 接口（返回 405 即代表彻底打通）：
```
[admin@aap-demo ~]$ kubectl exec -n redhat-rhaap-portal deployment/redhat-rhaap-portal -- curl -k -s -o /dev/null -w "%{http_code}\n" https://aap-aap-operator.apps-crc.testing/o/token/
Defaulted container "backstage-backend" out of: backstage-backend, ansible-devtools-server, install-dynamic-plugins (init)
405
```
```

如果返回值为405即OK。
