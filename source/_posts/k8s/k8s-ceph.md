---
title: Kubernete中使用ceph存储
---

## 环境配置

### 安装ceph-common

所有node上安装ceph-common

```bash
yum install -y ceph-common
```

### 配置ceph secret

在ceph monitor上执行

```bash
grep key /etc/ceph/ceph.client.admin.keyring |awk '{printf "%s", $NF}'|base64
```

将输出结果写入ceph-secret中

```
# cat ceph-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
type: "kubernetes.io/rbd"
data:
  key: QVFCTE1ETmE3ZThhS2hBQTZGRU55N1J0YVM1YU1Ddlo1VkF4dkE9PQ==
```

```bash
kubectl creat -f ceph-secret.yaml
```

### 配置rbac
```

# cat ceph-rbac.yaml
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["watch", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
subjects:
  - kind: ServiceAccount
    name: rbd-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: rbd-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl creat -f ceph-rbac.yaml
```

### 配置rbd provisioner

```
# cat ceph-provisioner.yaml
```
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rbd-provisioner
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "quay.io/external_storage/rbd-provisioner:v0.1.0"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccount: rbd-provisioner
```

```bash
kubectl creat -f ceph-provisioner.yaml
```

### 配置StorageClass

```
# cat ceph-sc.yaml
```
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: ceph
provisioner: ceph.com/rbd
parameters:
  monitors: 172.29.86.162:6789 # ceph monitor地址
  adminId: admin
  adminSecretName: ceph-secret
  pool: rbd
  userId: admin
  userSecretName: ceph-secret
```

## 测试
```
# cat ceph-pvc.yaml
```

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
 name: ceph-pvc
spec:
 accessModes:
    - ReadWriteOnce
 resources:
   requests:
     storage: 20Gi
 storageClassName: ceph
```

```bash
kubectl create -f  ceph-pvc.yaml
```

```
# cat cept-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-test
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: ceph-pvc
  containers:
  - name: ceph-test
    image: "busybox"
    command: ["/bin/sh", "-c", "sleep 36000"]
    volumeMounts:
    - name: vol
      mountPath: /vol
```
```bash
kubectl create -f  ceph-pod.yaml
```

```bash
kubectl exec -it ceph-test /bin/sh
# df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/mapper/docker-8:2-9699913-b0a2bdb12e4ab87811c759041fa335dbceaa1a999136efb99c0cc2a027e529a5
                         10.0G     34.2M     10.0G   0% /
tmpfs                   377.2G         0    377.2G   0% /dev
tmpfs                   377.2G         0    377.2G   0% /sys/fs/cgroup
/dev/rbd0                19.6G     44.0M     18.5G   0% /vol
/dev/sda2               218.0G     18.6G    188.3G   9% /dev/termination-log
/dev/sda2               218.0G     18.6G    188.3G   9% /etc/resolv.conf
/dev/sda2               218.0G     18.6G    188.3G   9% /etc/hostname
/dev/sda2               218.0G     18.6G    188.3G   9% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                   377.2G     12.0K    377.2G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                   377.2G         0    377.2G   0% /proc/kcore
tmpfs                   377.2G         0    377.2G   0% /proc/timer_list
tmpfs                   377.2G         0    377.2G   0% /proc/timer_stats
tmpfs                   377.2G         0    377.2G   0% /proc/sched_debug
``` 

## Debug
* 查看rbd-provisioner日志

```bash
kubectl log $(kubectl get pod | grep rbd-provisioner | awk '{print $1}')
```

*  查看pod所在node的kubelet日志

```bash
systemctl status kubelet.service -l
```

或者

```bash
journalctl  -r -u kubelet.service
```