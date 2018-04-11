---
title: Kubernete中使用solidfire存储
date: 2018-03-02 19:05:02
tags: 
- kubenretes
- storage
- solidfire
- san
---

## 安装iscsi客户端
所有node上安装
```bash
yum install iscsi-initiator-utils
```
```bash
systemctl enable iscsi
systemctl start iscsi
```
## 创建Volume Access Group（VAG）
登录web管理界面，添加Access Group "trident"

![enter image description here](https://keithtenzer.files.wordpress.com/2017/03/screen-shot-2017-03-16-at-11-32-48.png?w=440)

将node的iqn添加到trident组中，用下面命令查看iqn
```bash
cat /etc/iscsi/initiatorname.iscsi
```


## 配置Trident插件（仅在master上操作）

### 下载安装Trident插件
```bash
wget https://github.com/NetApp/trident/releases/download/v18.01.0/trident-installer-18.01.0.tar.gz
tar -xf trident-installer-18.01.0.tar.gz
cd trident-installer
```
编辑添加backend.json
```
vi setup/backend.json
```
编辑完成后内容如下：
```
cat backend.json
```
```json
{
    "version": 1,
    "storageDriverName": "solidfire-san",
    "Endpoint": "https://admin:root1234@172.29.101.251/json-rpc/7.0",
    "SVIP": "172.29.99.8:3260",
    "TenantName": "trident",
    "AccessGroups": [6],
    "InitiatorIFace": "default",
    "Types": [{"Type": "Bronze", "Qos": {"minIOPS": 1000, "maxIOPS": 2000, "burstIOPS": 4000}},
              {"Type": "Silver", "Qos": {"minIOPS": 4000, "maxIOPS": 6000, "burstIOPS": 8000}},
              {"Type": "Gold", "Qos": {"minIOPS": 6000, "maxIOPS": 8000, "burstIOPS": 10000}}]
}
```
这里的 TenantName 就是刚才创建的 VAG 的名称, AccessGroups 填 VAG 的 ID。


执行安装脚本
```bash
./install_trident.sh -n trident
```

等待镜像下载好，pod状态正常
```
# kubectl get pod -n trident
NAME                       READY     STATUS    RESTARTS   AGE
trident-7d5d659bd7-tzth6   2/2       Running   1          14s

# ./tridentctl -n trident version
+----------------+----------------+
| SERVER VERSION | CLIENT VERSION |
+----------------+----------------+
| 18.01.0        | 18.01.0        |
+----------------+----------------+
```

#### 添加存储后端
```bash
./tridentctl -n trident create backend -f setup/backend.json
```
#### 配置StorageClass
```
cat solidfire-gold-sc.yaml
```
```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: solidfire-gold
provisioner: netapp.io/trident
parameters:
  media: "hybrid"
  provisioningType: "thin"
  snapshots: "true"
  requiredStorage: "solidfire_172.29.99.8:Gold"
reclaimPolicy: Delete  
```
```bash
kubectl create -f  solidfire-gold-sc.yaml
```
### 测试

```
cat solidfire-pvc.yaml
```
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: solidfire-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: solidfire-gold
```

```bash
kubectl create -f  solidfire-pvc.yaml
```
```
cat solidfire-pod.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: solidfire-test
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: solidfire-pvc
  containers:
  - name: solidfire-test
    image: "busybox"
    command: ["/bin/sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: vol
      mountPath: /vol
```
```bash
kubectl create -f  solidfire-pod.yaml
```

```bash
kubectl exec -it solidfire-test /bin/sh
# df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/mapper/docker-8:2-9699913-7f0df876ba69f64045f74ec95d44c27f9b47a2cffb7b1beba781763dcc5f385e
                         10.0G     34.2M     10.0G   0% /
tmpfs                   377.2G         0    377.2G   0% /dev
tmpfs                   377.2G         0    377.2G   0% /sys/fs/cgroup
/dev/sdc                  1.2G      3.6M      1.1G   0% /vol
/dev/sda2               218.0G     15.7G    191.2G   8% /dev/termination-log
/dev/sda2               218.0G     15.7G    191.2G   8% /etc/resolv.conf
/dev/sda2               218.0G     15.7G    191.2G   8% /etc/hostname
/dev/sda2               218.0G     15.7G    191.2G   8% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                   377.2G     12.0K    377.2G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                   377.2G         0    377.2G   0% /proc/kcore
tmpfs                   377.2G         0    377.2G   0% /proc/timer_list
tmpfs                   377.2G         0    377.2G   0% /proc/timer_stats
```

# Debug
## 查看日志

```bash
./tridentctl -n trident logs
```

## 卸载插件

```bash
./uninstall_trident.sh -n trident -a
```

# 参考
* https://netapp-trident.readthedocs.io/en/stable-v18.01/kubernetes/deploying.html
* https://keithtenzer.com/2017/04/05/storage-for-containers-using-netapp-solidfire-part-vi/
