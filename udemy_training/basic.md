# Pod

containers
```yml
command: DockerfileのENTRYPOINTを上書き
args   : DockerfileのCMDを上書き

imagePullPolicy: IfNotPresent # default
command: ["sh", "-c"]
args:
- |
  echo "$(MESSAGE)"
env: [{name: "MESSAGE", value: "Hello World!"}]
volumeMounts:
- name: storage
  mountPath: /home/user
```

volume
```yml
spec.containers.volumeMounts.nameとspec.volumes.nameを一致させる

spec:
  ...
  volumes:
  - name: storage
    hostPath, nfs, configMap, secret, emptyDir: ...
------------------------------------------------------------     
hostPath:
  path: /../..
  type: Directory, DirectoryOrCreate, File, FileOrCreate

nfs:
  server: 192.394.392.101
  path: /nfs/data/tmp..

configMap:
  name: sample
  items:
  - key: sample.cfg
    path: sample.cfg

secret:
  secretName: sample-secret
  items:
  - key: keyfile
    path: keyfile

emptyDir: {}
------------------------------------------------------------     
```

pod.yml
```yml
apiVersion: v1
kind: Pod
metadata:
  name: sample
spec:
  contaienrs:
  - name: nginx
    image: nginx:1.17.2-alpine
    volumeMounts:
    - name: storage
      mountPath: /home/nginx
  volumes:
  - name: storage
    hostPath:
      path: /data/storage
      type: Directory
```

```s
$ touch /data/storage/message ; echo 'Hello World!' >> /data/storage/message
$ kubectl apply -f pod.yml
$ kubectl get po
$ kubectl exec -it pod/sample sh
> $ cat /data/nginx/message
    'Hello World!'
```



# ReplicaSet
replicaset.yml
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: sample
spec:
  replicas: 2
  selector:        # Pod/metadata.labalsに一致させる
    matchLabels:
      app: web
      env: study
  template: 
    metadata:
      name: nginx
      labels:
        app: web
        env: study
      spec:
        containers:
        - name: nginx
          image: nginx:1.17.2-alpine
```
```s
$ kubectl apply -f replicaset.yml
$ kubectl get po,rs
```



# Deployment
deployment.yml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  # annotations:
    # kubernetes.io/change-cause: "update nginx 1.17.3"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      env: study
  revisionHistoryLimit: 14
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      name: nginx
      labels:
        app: web
        env: study
    sepc:
      containers:
      - name: nginx
        image: ngixn:1.17.2-alpine
        # image: ngixn:1.17.3-alpine
```

```s
$ kubectl apply -f deploy.yml
$ kubectl get all
$ kubectl rollout history deploy/nginx
  REVISION    CHANGE-CAUSE
  1           none

<yml変更>

$ kubeclt apply -f deploy.yml
$ kubectl rollout history deploy/nginx
  REVISION    CHANGE-CAUSEn
  1           none
  2           Update nginx 1.17.3

<ロールバック>
$ kubectl rollout undo deploy/nginx # --to-revision=N 指定REVISIONに戻す
  AGEが若くなってる→何か変更されてるのが分かる
$ kubectl rollout history deploy/nginx
  REVISION    CHANGE-CAUSE
  2           Update nginx 1.17.3
  3           none
```



# Service
外部公開、名前解決、L4ロードバランサ―
```yml
type: ClusterIP, Nodeport, LoadBalancer, ExternalName
clusterIP: None(HeadlessService), ""(自動採番), "123.456.789.123"(指定IP)
ports:
- port: 80  # サービス受付ポート
  targetPort: 80  # コンテナ転送ポート
  nodePort: 8080  # ノード受付ポート
selector:
  app: web
  env: study
```

service.yml
```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
    env: study
spec:
  containers:
  - name: nginx
    image: nginx:1.17.2-alpine

---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector:
    app: web
    env: study
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30000
```

```s
$ kubectl apply -f service.yml
$ kubectl get all

<ブラウザ>
VMのIP:30000
```



# Config Map
specではなくdataにキーバリューで保存
・環境変数（valueFromに作成したconfigMapを指定）
・ファイル（volumesとvolumeMountsに指定）

config-map.yml
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-config
data:
  type: "application"  # 環境変数

  sample.cfg: |        # ファイル
    user: itsuki.miyake

---
apiVersion: v1
kind: Pod
metadata:
  name: sample
spec:
  containers:
  - name: sample
    image: nginx:1.17.2-alpine

    env:               # 環境変数
    - name: TYPE
      valueFrom:
        configMapKeyRef:
          name: sample-config
          key: type

    volumeMounts:       # ファイル
    - name: config-storage
      mountPath: /home/nginx
  volumes:
  - name: config-storage
    configMap:
      name: sample-config
      items:
      - key: sample.cfg
        path: sample.cfg
```

VM
```s
$ kubectl apply -f config-map.yml
$ kubectl get po
$ kubectl exec -it sample sh
> $ env                 # 環境変数
    ...
    TYPE=application
    ...

> $ ls /home/nginx      # ファイル
    sample.cfg
> $ cat /home/nginx/sample.cfg
    user: itsuki.miyake
```


# Secret
base64でエンコードされた機微情報
specではなくdataにキーバリューで保存
（キーバリューのバリューがエンコードされた文字列）

・コマンドで直接生成　→　キーバリューは引数で複数指定
・マニフェストファイルから生成　→　base64の取得必要

コマンド生成
```s
$ touch keyfile ; echo "YOUR-SECRET-KEY" >> keyfile
$ kubectl create secret generic sample-secret \
> --from-literal=message='Hello World !' \
> --from-file=./keyfile
$ kubectl get secret
$ kubectl get secret/sample-secret -o yaml
  apiVersion: v1
  data:
    keyfile: fkldsjafkkkkjkj..
    message: fjodiewjfjsaifm..
  kind: Secret
  ...
```

マニフェストファイル生成
```s
$ echo -n 'Hello World !' | base64
  jfdosafijsafj;ldsa...コピペ
$ echo -n 'YOUR-SECRET-KEY' | base64
  oifwajfjiowaksjfio...コピペ
``` 
secret.yml
```yml
apiVersion: v1
kind: Secret
emtadata:
  name: sample-secret
data:
  message: jfdosafijsafj;ldsa...
  keyfile: oifwajfjiowaksjfio...

---
apiVersion: v1
kind: Pod
metadata:
  name: sample
spec:
  containers:
  - name: sample
    image: nginx:1.17.2-alpine
    env:
    - name: MESSAGE
      volueFrom:
        secretKeyRef:
          name: sample-secret
          key: message
    volumeMounts:
    - name: secret-storage
      mountPath: /home/nginx
  volumes:
  - name: secret-storage
    secret:
      secretName: sample-secret
      items:
      - key: keyfile
        path: keyfile
```

```s
$ kubectl apply -f secret.yml
$ kubectl get po
$ kubectl exec -it sample sh
> $ ls /home/nginx/
    keyfile
> $ cat /home/nginx/keyfile
    YOUR-SECRET-KEY    # base64で保存されたが接続先では正しく展開されている
> $ env
    ...
    MESSAGE=Hello World !
```


# 永続データ
・PersistentVolume(PV)        永続データの実態
・PersistentVolumeClaim(PVC)  永続データの要求

storage.yml
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volume-01
spec:
  storageClassName: slow
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/storage
    type: Directory

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: volume-claim
spec:
  storageClassName: slow
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```s
$ ls /data
  storage
$ kubectl apply -f storage.yml
$ kubectl get pvc,pv
```


# StatefulSet
deploymentとほぼ一緒

statefulset.yml
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volume-01
spec:
  strateClassName: slow
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/storage
    type: Directory

---
apiVersion: v1
kind: Service
metadata:
  name: sample-svc
spec:
  clusterIP: None
  selector:
    app: web
    env: study
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      env: study
  revisionHistoryLimit: 14
  serviceName: sample-svc
  template:
    metadata:
      name: nginx
      labels:
        app: web
        env: study
      spec:
        containers:
        - name: nginx
          image: nginx1.17.2-alpine
          volumeMounts:
          - name: storage
            mountPath: home/nginx
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      storageClassName: slow
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 1Gi
```

```s
$ ls /data
  storage
$ kubectl apply -f statefulset.yml
$ kubeclt get all
  ... pod/nginx-0   1/1  Running   0   39s

# centosをデバッグ用で起動
$ kubectl run debug --image=centos:7 -it --rm --restart=Never -- sh
> $ curl http://nginx-0.sample-svc/
```



# Ingress
外部公開、L7ロードバランサ―（URLでサービスを切り替えられる）

ingress.yml
```yml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
    env: study
  ports:
  - port: 80
    targetPort: 80
  # defaltのclusterIP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      env: study
  template:
    metadata:
      name: nginx
      labels:
        app: web
        env: study
    spec:
      contaienrs:
      - name: nginx
        image: nginx:1.17.2-alpine

---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: frontend
  annotations:
    kubernetes.io/ingress.class: "nginx" # minikubeの場合
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: web-svc
          servicePort: 80
```

```s
$ kubectl apply -f ingress.yml
$ kubectl get ing,svc,deploy
  ingressにIPが割り振られていれば外部と通信できる

<ブラウザ>
ingressのIP
```