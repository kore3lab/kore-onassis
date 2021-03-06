[TOC]

## K8S master node 구성
 * 각 master node에서 control plane 구성에 필요한 TLS 인증서를 생성한다.
 * /etc/kubernetes/pki/apiserver.key파일이 존재하면 이미 인증서가 생성되어 있다고 가정하고 skip한다.
 * kubectl, kubelet, kubeadm package를 설치하고 ubunutu의 경우 hold로 mark함으로 apt upgrade시 자동 upgrade되는 것을 방지한다.
 * jq 명령 설치
 * audit policy, secret encryption 구성
 * kubeadm.yaml 파일 구성
 * /etc/kubelet/kubelet 설정 파일 구성
 * kubelet 기동
 * kubeadm init으로 초기 master 구성
 * admin.conf, control-manager.conf, scheduler.conf, acloud-client-kubeconfig 파일 구성
 * .profile or .bash_profile 구성
 * Calico, Metrics server 설치

### 개별 master node TLS 인증서 생성
 
```bash
$ mkdir /etc/kubernetes/acloud 

$ openssl genrsa -out /etc/kubernetes/pki/apiserver.key 2048
$ openssl req -new -key /etc/kubernetes/pki/apiserver.key -subj '/CN=kube-apiserver' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/pki/apiserver.crt -days 36500 -extensions v3_req_apiserver -extfile /opt/kubernetes/pki/common-openssl.conf

$ openssl genrsa -out /etc/kubernetes/pki/apiserver-kubelet-client.key 2048
$ openssl req -new -key /etc/kubernetes/pki/apiserver-kubelet-client.key -subj '/CN=kube-apiserver-kubelet-client/O=system:masters' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/pki/apiserver-kubelet-client.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf

$ openssl genrsa -out /etc/kubernetes/pki/front-proxy-client.key 2048
$ openssl req -new -key /etc/kubernetes/pki/front-proxy-client.key -subj '/CN=front-proxy-client' |
  openssl x509 -req -CA /etc/kubernetes/pki/front-proxy-ca.crt -CAkey /etc/kubernetes/pki/front-proxy-ca.key -CAcreateserial -out /etc/kubernetes/pki/front-proxy-client.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf

$ openssl genrsa -out /etc/kubernetes/pki/admin.key 2048
$ openssl req -new -key /etc/kubernetes/pki/admin.key -subj '/O=system:masters/CN=kubernetes-admin' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/pki/admin.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf

$ openssl genrsa -out /etc/kubernetes/pki/controller-manager.key 2048
$ openssl req -new -key /etc/kubernetes/pki/controller-manager.key -subj '/CN=system:kube-controller-manager' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/pki/controller-manager.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf

$ openssl genrsa -out /etc/kubernetes/pki/scheduler.key 2048
$ openssl req -new -key /etc/kubernetes/pki/scheduler.key -subj '/CN=system:kube-scheduler' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/pki/scheduler.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf

// 아래 acloud 인증서는 master1에서만 작업하면 됨.
$ openssl genrsa -out /etc/kubernetes/acloud/acloud-client.key 2048
$ openssl req -new -key /etc/kubernetes/acloud/acloud-client.key -subj '/CN=acloud-client' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/acloud/acloud-client.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf
```

 * Ubuntu 
```bash
$ apt-get install -y kubelet=1.20.8-00 kubeadm=1.20.8-00 kubectl=1.20.8-00
$ apt-get install -y jq

```  
 * Centos, RHEL 
```bash
$ yum clean all; yum -y update
$ yum install -y kubelet-1.20.8 kubeadm-1.20.8 kubectl-1.20.8
$ yum install -y jq
```  

### Audit policy
```bash
$ cat > /etc/kubernetes/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: "" # core
        resources: ["endpoints", "services", "services/status"]
  - level: None
    users: ["system:unsecured"]
    namespaces: ["kube-system"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["configmaps"]
  - level: None
    users: ["kubelet"] # legacy kubelet identity
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    users:
      - system:kube-controller-manager
      - system:kube-scheduler
      - system:serviceaccount:kube-system:endpoint-controller
    verbs: ["get", "update"]
    namespaces: ["kube-system"]
    resources:
      - group: "" # core
        resources: ["endpoints"]
  - level: None
    users: ["system:apiserver"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["namespaces", "namespaces/status", "namespaces/finalize"]
  - level: None
    users:
      - system:kube-controller-manager
    verbs: ["get", "list"]
    resources:
      - group: "metrics.k8s.io"
  - level: None
    nonResourceURLs:
      - /healthz*
      - /version
      - /swagger*
  - level: None
    resources:
      - group: "" # core
        resources: ["events"]
  - level: Request
    users: ["kubelet", "system:node-problem-detector", "system:serviceaccount:kube-system:node-problem-detector"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
  - level: Request
    userGroups: ["system:nodes"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
  - level: Request
    users: ["system:serviceaccount:kube-system:namespace-controller"]
    verbs: ["deletecollection"]
  - level: Metadata
    resources:
      - group: "" # core
        resources: ["secrets", "configmaps"]
      - group: authentication.k8s.io
        resources: ["tokenreviews"]
  - level: Request
    verbs: ["get", "list", "watch"]
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "scheduling.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
  - level: RequestResponse
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "scheduling.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
  # Default level for all other requests.
  - level: Metadata
EOF
```

### Secret Encryption
```bash
$ cat > /etc/kubernetes/secrets_encryption.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: cocktail
          secret: "N9o5U/W/evtItR6rUYogsh8C2x1Fre7/s4WSzmHXC7k="
    - identity: {}
EOF
```

### kubeadm.yaml
```bash
$ cat > /etc/kubernetes/kubeadm.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.77.121               // 각 master node IP
  bindPort: 6443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
etcd:
  external:
    endpoints:
    - https://192.168.77.121:2379
    - https://192.168.77.122:2379
    - https://192.168.77.123:2379        
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/etcd/server.crt
    keyFile: /etc/kubernetes/pki/etcd/server.key
networking:
  dnsDomain: cluster.local
  serviceSubnet: 172.20.0.0/16
  podSubnet: 10.0.0.0/16
kubernetesVersion: 1.20.8
controlPlaneEndpoint: 192.168.77.121:6443      // master1 node IP or LB IP
certificatesDir: /etc/kubernetes/pki
apiServer:
  extraArgs:
    bind-address: "0.0.0.0"
    apiserver-count: "1"
    secure-port: "6443"
    feature-gates: TTLAfterFinished=true,RemoveSelfLink=false
    default-not-ready-toleration-seconds: "30"
    default-unreachable-toleration-seconds: "30"
    encryption-provider-config: /etc/kubernetes/secrets_encryption.yaml
    audit-log-maxage: "7"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    audit-log-path: /var/log/kubernetes/kubernetes-audit.log
    audit-policy-file: /etc/kubernetes/audit-policy.yaml
  extraVolumes:
  - name: audit-policy
    hostPath: /etc/kubernetes
    mountPath: /etc/kubernetes
    pathType: DirectoryOrCreate
    readOnly: true
  - name: k8s-audit
    hostPath: /var/log/kubernetes
    mountPath: /data/k8s-audit
    pathType: DirectoryOrCreate
  certSANs:
  - 192.168.77.121                            // master1 node IP or LB IP
  - localhost
  - 127.0.0.1
controllerManager:
  extraArgs:
    address: "0.0.0.0"
    node-monitor-period: 2s
    node-monitor-grace-period: 16s
    feature-gates: TTLAfterFinished=true,RemoveSelfLink=false
scheduler:
  extraArgs:
    address: "0.0.0.0"
    feature-gates: TTLAfterFinished=true,RemoveSelfLink=false
imageRepository: 192.168.77.128/google_containers
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates: { TTLAfterFinished: true }
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
nodeStatusUpdateFrequency: 4s
readOnlyPort: 0
clusterDNS:
- 172.20.0.10
EOF
```

### kubelet 설정 및 기동
```bash
$ cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS="--log-dir=/data/log \
--logtostderr=false \
--v=2 \
--cluster-dns=172.16.0.10 \
--cluster-domain=cluster.local \
--node-status-update-frequency=4s \
--node-labels=cube.acornsoft.io/role=master,cube.acornsoft.io/clusterid=test-cluster"
EOF

$ systemctl enable kubelet
$ systemctl start kubelet

```

### kubeadm init 으로 master node 초기화
```bash
$ kubeadm init --config=/etc/kubernetes/kubeadm.yaml --ignore-preflight-errors=all
```


### admin.conf, controller-manager.conf, scheduler.conf 파일 구성 - 인증서 기입 필요
```bash
$ cat > /etc/kubernetes/admin.conf <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $(cat /etc/kubernetes/pki/ca.crt | base64 -w0)
    server: https://192.168.77.121:6443            // 각 master node ip로 설정
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: $(cat /etc/kubernetes/admin.crt | base64 -w0)
    client-key-data: $(cat /etc/kubernetes/admin.key | base64 -w0)
EOF

$ cat > /etc/kubernetes/controller-manager.conf <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $(cat /etc/kubernetes/pki/ca.crt | base64 -w0)
    server: https://192.168.77.121:6443             // 각 master node ip로 설정
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-controller-manager
  name: system:kube-controller-manager@kubernetes
current-context: system:kube-controller-manager@kubernetes
kind: Config
preferences: {}
users:
- name: system:kube-controller-manager
  user:
    client-certificate-data: $(cat /etc/kubernetes/controller-manager.crt | base64 -w0)
    client-key-data: $(cat /etc/kubernetes/controller-manager.key | base64 -w0)
EOF

$ cat > /etc/kubernetes/scheduler.conf <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $(cat /etc/kubernetes/pki/ca.crt | base64 -w0)
    server: https://192.168.77.121:6443             // 각 master node ip로 설정
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-scheduler
  name: system:kube-scheduler@kubernetes
current-context: system:kube-scheduler@kubernetes
kind: Config
preferences: {}
users:
- name: system:kube-scheduler
  user:
    client-certificate-data: $(cat /etc/kubernetes/scheduler.crt | base64 -w0)
    client-key-data: $(cat /etc/kubernetes/scheduler.crt | base64 -w0)
EOF
```

### acloud-client-kubeconfig 파일 구성
```bash
$ openssl genrsa -out /etc/kubernetes/acloud/acloud-client.key 2048
$ openssl req -new -key /etc/kubernetes/acloud/acloud-client.key -subj '/CN=acloud-client' |
  openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out /etc/kubernetes/acloud/acloud-client.crt -days 36500 -extensions v3_req_client -extfile /opt/kubernetes/pki/common-openssl.conf

$ cat > /etc/kubernetes/acloud/acloud-client-crb.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: acloud-binding
subjects:
- kind: User
  name: acloud-client
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF

$ cat > /etc/kubernetes/acloud/acloud-client-kubeconfig <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $(cat /etc/kubernetes/pki/ca.crt | base64 -w0)
    server: https://192.168.77.121:6443             // 각 master node ip로 설정
  name: acloud-client
contexts:
- context:
    cluster: acloud-client
    user: acloud-client
  name: acloud-client
current-context: acloud-client
kind: Config
preferences: {}
users:
- name: acloud-client
  user:
    client-certificate-data: $(cat /etc/kubernetes/acloud/acloud-client.crt | base64 -w0)
    client-key-data: $(cat /etc/kubernetes/acloud/acloud-client.key | base64 -w0)
EOF
```

### alias
 * RHEL, Centos .bash_profile 설정
```bash
$ cat > /root/.bash_profile1 <<EOF
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/peer.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/peer.key

export KUBECONFIG=/etc/kubernetes/admin.conf

alias ll='ls -al'
alias etcdlet="etcdctl --endpoints=https://192.168.77.121:2379,https://192.168.77.122:2379,https://192.168.77.123:2379 "
alias psg="ps -ef | grep "
alias wp="watch -n1 'kubectl get pods --all-namespaces -o wide'"
alias k="kubectl "
EOF
```

 * Ubuntu .profile 설정
```bash
$ cat > /root/.profile <<EOF
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/peer.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/peer.key

export KUBECONFIG=/etc/kubernetes/admin.conf

alias ll='ls -al'
alias etcdlet="etcdctl --endpoints=https://192.168.77.121:2379,https://192.168.77.122:2379,https://192.168.77.123:2379 "
alias psg="ps -ef | grep "
alias wp="watch -n1 'kubectl get pods --all-namespaces -o wide'"
alias k="kubectl "
alias cert="openssl x509 -text -noout -in "
EOF
```

### alias
 * master isolation 옵션이 있을 경우 taint 구성
```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf taint nodes vm-onassis-01 node-role.kubernetes.io/master:NoSchedule-
kubectl --kubeconfig=/etc/kubernetes/admin.conf taint nodes vm-onassis-02 node-role.kubernetes.io/master:NoSchedule-
kubectl --kubeconfig=/etc/kubernetes/admin.conf taint nodes vm-onassis-03 node-role.kubernetes.io/master:NoSchedule-
```

### calco and metrics-server copy 및 변수 치환
```bash
$ scp calico.yaml root@192.168.77.121:/etc/kubernetes/addon/calico/calico.yaml
$ scp metrics-server-rbac.yaml.j2 root@192.168.77.121:/etc/kubernetes/addon/metrics-server/metrics-server-rbac.yaml
$ scp metrics-server-controller.yaml.j2 root@192.168.77.121:/etc/kubernetes/addon/metrics-server/metrics-server-controller.yaml
```

### metrics-server를 master node에 배포하기 위해서 tolerations 설정을 해준다.
- metrics-server-controller.yaml 내용을 아래와 같이 수정 한다.
  - 모든 노드에 스케줄 될수 있도록 tolerations 설정을 한다.
  - nodeSelector 또는 nodeAffinity 를 사용 해서 master node를 지정 한다.
  - replicaset을 증가시켜리면 master 노드는 ha 구성 되어 있어야 한다.
```yaml
spec:
  replicas: 2

    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                k8s-app: metrics-server
            topologyKey: kubernetes.io/hostname
      tolerations:
      # Make sure metrics-server gets scheduled on all nodes.
      - effect: NoSchedule
        operator: Exists
      nodeSelector:
        node-role.kubernetes.io/master: ''
```

- 전체 yaml 내용
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:aggregated-metrics-reader
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
rules:
- apiGroups:
    - ""
  resources:
    - pods
    - nodes
    - nodes/stats
    - namespaces
  verbs:
    - get
    - list
    - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    kubernetes.io/name: "Metrics-server"
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
---
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                k8s-app: metrics-server
            topologyKey: kubernetes.io/hostname
      tolerations:
      # Make sure metrics-server gets scheduled on all nodes.
      - effect: NoSchedule
        operator: Exists
      nodeSelector:
        node-role.kubernetes.io/master: ''
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      containers:
      - name: metrics-server
        image: "192.168.77.128/google_containers/metrics-server:v0.3.6"
        imagePullPolicy: Always
        command:
          - /metrics-server
          - --cert-dir=/tmp
          - --secure-port=4443
          - --metric-resolution=30s
          - --kubelet-preferred-address-types=InternalIP,InternalDNS,Hostname,ExternalIP,ExternalDNS
          - --kubelet-insecure-tls
          - --profiling=false
        ports:
        - containerPort: 4443
          name: https
        livenessProbe:
          httpGet:
            path: /healthz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
        readinessProbe:
          httpGet:
            path: /healthz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["all"]
          readOnlyRootFilesystem: true
          runAsGroup: 10001
          runAsNonRoot: true
          runAsUser: 10001
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      volumes:
      - name: tmp-dir
        emptyDir: {}
```

### Update apiserver endpoint in kube-proxy configmap when haproxy used as internal loadbalancer
```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get cm kube-proxy -n kube-system -o yaml | sed 's#server:.*#server: https://localhost:6443#g' | kubectl --kubeconfig=/etc/kubernetes/admin.conf apply -f -
kubectl --kubeconfig=/etc/kubernetes/admin.conf delete pods -n kube-system -l k8s-app=kube-proxy
```