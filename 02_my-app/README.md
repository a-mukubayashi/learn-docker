# my-app

## 基本的な使い方

- リソースの取得

```
kubectl get pod myapp
```

- IP アドレスや Node 情報取得

```
kubectl get pod myapp --output wide
# yaml形式
kubectl get pod myapp --output yaml
# json形式
kubectl get pod myapp --output json
```

`jq` と組み合わせて活用する

```
kubectl get pod myapp --output json | jq '.spec.containers[].image'
```

- ログレベルを変更

```
kubectl get pod myapp --v=7
```

https://kubernetes.io/ja/docs/concepts/cluster-administration/system-logs/#ログのverbosityレベル

- リソースの詳細を取得

```
kubectl describe pod myapp
```

Events の内容などは一定期間で消える

- コンテナのログを取得

```
kubectl logs myapp
# コンテナ指定
kubectl logs myapp --container hello-server
```

- 特定の Deployment に紐づく Pod ログ

```
kubectl logs deploy/hello-server
```

- ラベルを指定して参照する

```
# podの詳細
kubectl get pod --selector app=myapp
# ログ
kubectl logs --selector app=myapp
```

- デバッグ用のサイドカーコンテナを立ち上げる

```
kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.0 --target=hello-server -- sh
# コンテナ内
~ $ curl localhost:8080
Hello, world!~
```

- コンテナを即座に実行
  - `--rm` 実行が完了したら Pod を削除
  - `-stdin(-i)` オプションで標準入力に渡す
  - `--tty(-t)` オプションで擬似端末を割り当てる
  - `--restart=Never` Pod の再起動ポリシーを Never に設定。コンテナが終了しても再起動しない。デフォルトでは常に再起動するため、一度コマンド実行するようなケースでは設定を追加
  - `--command --` `--` の後に渡される拡張引数の`つ目が引数ではなくコマンドとして使われる

```
kubectl run busybox --image=busybox:1.36.1 --rm --stdin --tty --restart=Never --command -- nslookup google.com
```

`--stdin` と `--tty` は 2 つ同時に省略して `-it` と書く

- コンテナにログインする

```
# ログイン用のPodを作成
kubectl run curlpod --image=curlimages/curl:8.4.0 --command -- /bin/sh -c "while true;do sleep initify; done;" pod/curlpod created

❯ kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
curlpod   1/1     Running   0          18s

❯ kubectl get pod myapp --output wide
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
myapp   1/1     Running   0          66m   10.244.0.30   kind-control-plane   <none>           <none>

# ログイン
kubectl exec --stdin --tty curlpod -- /bin/sh
# curl <myapp podのip>:8080
~ $ curl 10.244.0.30:8080
Hello, world!~
```

- port-forward でアプリケーションにアクセス

pod には kubernetes クラスタ内用の IP アドレスが割り当てられる。そのため、何もしないとクラスタ外からのアクセスができない。

```
 kubectl port-forward myapp 5555:8080
 # 別のターミナル
 curl localhost:5555
 Hello, world!
```

- マニフェストの編集
  - あまり使わない/新しくマニフェストを作り kubectl apply が正規の手順とする

```
kubectl edit pod myapp
# 変更されてるか確認
kubectl get pod myapp --output yaml
```

- リソースを削除する
  - kubectl には Pod を再起動するというコマンドがないため、kubectl delete で代替え

```
❯ kubectl delete pod myapp
pod "myapp" deleted
```

Deployment を利用した Pod をすべて順番に再起動したい場合は `kubectl rollout restart`

## トラブルシューティング

- pod が動くことを確認

```
❯ kubectl apply -f my-app/myapp.yaml
pod/myapp created

❯ kubectl apply -f my-app/pod-destruction.yaml
pod/myapp configured

❯ k get pods
NAME    READY   STATUS             RESTARTS        AGE
myapp   0/1     CrashLoopBackOff   0 (2m15s ago)   12m

kubectl describe pod myapp
# Failed Pulling image "blux2/hello-server:1.1"と表示されるためeditして直す
# 参考：https://hub.docker.com/r/blux2/hello-server/tags

❯ kubectl edit pod myapp
pod/myapp edited

❯ k get pods
NAME    READY   STATUS    RESTARTS       AGE
myapp   1/1     Running   1 (4m3s ago)   13m

❯ k delete -f my-app/pod-destruction.yaml
pod "myapp" deleted

❯ k get pod myapp
Error from server (NotFound): pods "myapp" not found

--> ok
```

マニフェストに記述されているリソースが既に存在する場合で、設定値になんらかの更新があったときは「configured」と表示される

## Plugins

- [stern/stern](stern/stern)
  - ログの出力
- [derailed/k9s](https://github.com/derailed/k9s)
  - CLI ツール

```

k9s --context kind

```

- kubectx
  - context スイッチ
- kubens
  - namespace スイッチ

## 参考

- [kubectl チートシート](https://kubernetes.io/ja/docs/reference/kubectl/cheatsheet/)
