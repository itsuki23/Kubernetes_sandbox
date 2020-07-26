# Step6

### コントローラー
- ただのpod
  異常終了しても起動しない。
  コンテナ終了後削除必要。
  水平スケールできない。 
  <br>
- deployment
  webサーバーやappサーバーのように継続稼働し続けるタイプ
  <br>
- job
  実行したら終了するバッチ処理
```
podのみ
$ kubectl run webserver --image=nginx -d

deployment
$ kubectl create deploy webserver --image=nginx
$ kubectl scale --replicas=5 deploy/webserver

job
$ kubectl create job webserver --image=nginx
```



### コマンドオプション（非推奨になったので読み間違えないように）
- --restart=Never  -> Podのみ
- --restart=Always -> deployment