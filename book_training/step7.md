# Step7


## マニフェスト
step7/nginx-pod.yml
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

```
$ kubectl apply -f nginx-pod.yml

$ kubectl get po nginx -o wide
------------------------------
NAME                READY   STATUS      RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
nginx               1/1     Running     0          86s   10.244.2.18   node2   <none>           <none>

$ curl -m 3 http://10.244.2.18/
```

もう一つのコンテナから疎通確認
```
$ kubectl run busybox --image=busybox --restart=Never -it sh
$ wget -q -O - http://10.244.2.18/

$ exit
$ kubectl get po -o wide
------------------------------
NAME                READY   STATUS      RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
busybox             0/1     Completed   0          38s   10.244.1.7    node1   <none>           <none>
nginx               1/1     Running     0          12m   10.244.2.18   node2   <none>           <none>


疎通確認OK
busybox: 10.244.1.7 -> nginx: 10.244.2.18
```
<br><br><br>

## ヘルスチェック
ノード常駐のkubeletがコンテナのヘルスチェックを担当
#### プローブ
```
活性   プローブ(Liveness  Probe): 実行中かどうかをチェック。失敗したらコンテナを強制終了、再スタート。
準備状態プローブ(Readiness Probe): リクエストを受けられるかをチェック。失敗したらサービスからのトラフィックを止める。
```
#### プローブに対応するハンドラー
```
exec     : コンテナ内のコマンド実行。exitコード０で成功。それ以外失敗。
tcpSocket: 指定TCPポートに接続できれば成功。
httpGet  : 指定ポートとパスにHTTP GETが定期実行される。HTTPステータスコード200〜400で成功。
```

#### ファイル構成
```
step7/healthcheck
├── webapl
│   ├── Dockerfile
│   ├── package.json
│   └── webapl.js
└── webapl-pod.yml
```
webapl-pod.yml
```YML
apiVersion: v1
kind: Pod
metadata:
  name: webapl
spec:
  containers:
  - name: webapl
    image: maho/webapl:0.1
    livenessProbe:
      httpGet:
        path: /healthz
        port: 3000
      initialDelaySeconds: 3  # 探査開始までの猶予時間
      periodSeconds: 5        # チェック間隔
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 6
```
<br>

webapl.js
```js
const express = require('express')
const app = express()
var start = Date.now()

// Livenessプローブのハンドラー
// 模擬障害として、起動から40秒を超えると、HTTP 500内部エラーを返します。
// 40秒までは、HTTP 200 OKを返します。
// つまり、40秒を超えると、Livenessプローブが失敗して、コンテナが再起動します。
//
app.get('/healthz', function(request, response) {
    var msec = Date.now() - start
    var code = 200
    if (msec > 40000 ) {
	code = 500
    }
    console.log('GET /healthz ' + code)
    response.status(code).send('OK')
})

// Redinessプローブのハンドラー
// アプリケーションの初期化をシミュレーションして、
// 起動してから20秒経過後、HTTP 200を返します。 
// それまでは、HTTPS 200 OKを返します。
app.get('/ready', function(request, response) {
    var msec = Date.now() - start
    var code = 500
    if (msec > 20000 ) {
	code = 200
    }
    console.log('GET /ready ' + code)
    response.status(code).send('OK')
})
...
```

```
$ kubectl apply -f webapl-pod.yml
$ kubectl get po
$ kubectl logs webapl
------------------------------
GET /healthz 200
GET /healthz 200
GET /ready 500
GET /healthz 200
GET /ready 200
GET /healthz 200
GET /ready 200
...

$ kubectl describe po webapl
------------------------------
podの詳細表示（リクエストと失敗した等の流れも記載）
...
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  4m30s                default-scheduler  Successfully assigned default/webapl to node2
  Normal   Pulling    4m30s                kubelet, node2     Pulling image "maho/webapl:0.1"
  Normal   Pulled     4m23s                kubelet, node2     Successfully pulled image "maho/webapl:0.1"
  Normal   Created    93s (x3 over 4m23s)  kubelet, node2     Created container webapl
  Normal   Started    93s (x3 over 4m23s)  kubelet, node2     Started container webapl
  Normal   Pulled     93s (x2 over 2m58s)  kubelet, node2     Container image "maho/webapl:0.1" already present on machine
  Warning  Unhealthy  74s (x3 over 4m8s)   kubelet, node2     Readiness probe failed: HTTP probe failed with statuscode: 500
  Warning  Unhealthy  38s (x9 over 3m38s)  kubelet, node2     Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    38s (x3 over 3m28s)  kubelet, node2     Container webapl failed liveness probe, will be restarted
```
<br><br><br>


## 初期化コンテナ

- メインのコンテナ実行前にinitコンテナ実行。

step7/init/init-sample.yml
```YML
apiVersion: v1
kind: Pod
metadata:
  name: init-sample
spec:
  containers:
  - name: main
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "tail -f /dev/null"]
    volumeMounts:
    - mountPath: /docs  # 共有ボリュームのマウントポイント
      name: data-vol
      readOnly: false

  # ストレージの中に新規ディレクトリ作成し、オーナー変更してデータ取り込む
  initContainers:
  - name: init
    image: alpine
    command: ["/bin/sh"]
    args: ["-c", "mkdir /mnt/html; chown 33:33 /mnt/html"]
    volumeMounts:
    - mountPath: /mnt  # 共有ボリュームのマウントポイント
      name: data-vol
      readOnly: false
  
  volumes:             # ポッド上の共有ボリューム
  - name: data-vol
    emptyDir: {}
```
<br><br><br>

## サイドカーパターン

/step7/sidecar/contents-cloner: GitHubから定期的にpullするシェルスクリプト
```sh
#!/bin/bash
# 最新webコンテンツをGitHubからコンテナへ取り込む

# コンテンツ元の環境変数がなければエラー終了
if [ -z $CONTENTS_SOURCE_URL ]; then
    exit 1
fi

# 初回はGitHubからコンテンツをクローン
git clone $CONTENTS_SOURCE_URL /data

# ２回目以降は1分ごとに変更差分を取得する
cd /data
while ture
do
    data
    sleep 60
    git pull
done
```

/step7/sidecar/Dockerfile: nginxのコンテナ
```docker
## Contents Cloner Image
FROM ubuntu:16.04
RUN apt-get update && apt-get install -y git
COPY ./contents-cloner /contents-cloner
RUN chmod a+x /contents-cloner
WORKDIR /
CMD ["/contents-cloner"]
```

/step7/sidecar/webserver.yml: マニフェスト
```yml
## サイドカー構成

apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: contents-vol
      readOnly: true

  - name: cloner
    image: maho/c-cloner:0.1
    env:
    - name: CONTENTS_SOURCE_URL
      value: "https://github.com/takara9/web-contents"  # この値を変えるだけなので再利用性が高い
    volumeMounts:
    - mountPath: /data
      name: contents-vol
      
  volumes:
  - name: contents-vol
    emptyDir{}
```

#### 実行
- GitHubからpullしたものを反映、表示。
- 内容を変更してpushしても、数分後には自動でpullして反映。
```
$ kubectl apply -f webserver.yml
$ kubectl get po -o wide

$ kubectl run busybox --image=busybox --restart=Never --rm -it sh
$ wget -q -O - http://172.....

$ kubectl delete po webserver
```


