# replicaset

## ReplicaSet

- ReplicaSet を作成
  - 指定した数の pod を複製

```
❯ kubectl apply -f replicaset.yaml
replicaset.apps/httpserver created

❯ k get pod
NAME               READY   STATUS    RESTARTS   AGE
httpserver-2hzmg   1/1     Running   0          28s
httpserver-kcsdj   1/1     Running   0          28s
httpserver-m2lpv   1/1     Running   0          28s

❯ k delete replicaset httpserver
replicaset.apps "httpserver" deleted
```

## Deployment

Deployment は ReplicaSet を管理する、より上位レベルの概念

```
❯ kubectl apply -f replicaset/deployment.yaml
deployment.apps/nginx-deployment created

❯ k get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           5m35s

❯ k get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-c48b99c84   3         3         3       5m58s

# 編集する場合
❯ k apply -f replicaset/deployment.yaml
deployment.apps/nginx-deployment configured

❯ k get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-c48b99c84   0         0         0       7m46s
nginx-deployment-d67949546   3         3         3       18s <- 新しいReplicaSet
# imageが新しくなっているか確認
❯ k get deployment nginx-deployment -o=jsonpath='{.spec.template.spec.containers[0].image}'
nginx:1.25.3

# 新規バージョン追加時の挙動確認
❯ kubectl describe deployment nginx-deployment
...
StrategyType:           RollingUpdate -- 更新の方法を指定
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
```

RollingUpdateStrategy は Pod にも ReplicaSet にもない、Deployment のみに存在するフィールド

- StrategyType・・・Pod 更新の挙動設定
  - Recreate
    - 全部の Pod を同時に更新
  - RollingUpdate - 順番に更新

RollingUpdate は Kubernetes 固有の単語ではなく一般的な単語。これでアプリケーションのアップデート中もサービスの停止を最小限に抑えられる。

- Deployment のローリングアップデート
  - `.spec.strategy.type==RollingUpdate` と指定されているとき、Deployment はローリングアップデートにより Pod を更新
    - maxUnavailable
      - 最大いくつの Pod を同時にシャットダウンできるか
      - default: 25% は Pod 全体の 25%まで同時シャットダウン可能
    - maxSurge
      - 最大いくつの Pod を新規作成するか

[deployment のローリングアップデート](https://kubernetes.io/ja/docs/concepts/workloads/controllers/deployment/#deploymentのローリングアップデート)

- `get pod` の結果を監視

```
kubectl get pod --watch
```

- `strategy.type=Recreate` で試す

```
# 別のターミナル
❯ kubectl get pod --watch

❯ k apply -f deployment-recreate.yaml
deployment.apps/nginx-deployment configured
```

watch したターミナルで Pod が一気に Terminating ->Pending->ContainerCreating->Running に遷移

Recreate は同時にすべての Pod を再作成するため速度は速い。その分、アプリケーションが一旦接続不能になる。

- `strategy.type=RollingUpdate` で試す

```
❯ k apply -f deployment-rollingupdate.yaml
deployment.apps/nginx-deployment created

# max surgeが100%(DesiredなPodを一度に全部新規作成)か確認
❯ k get deployment nginx-deployment -o jsonpath='{.spec.strategy}'
{"rollingUpdate":{"maxSurge":"100%","maxUnavailable":"25%"},"type":"RollingUpdate"}
```

watch しているターミナルで順番に Pod が入れ替わる。

max surge が 100%で Pod の更新が最も早くて安全だが、必要なリソースが倍になる（更新時に Pod が倍になる）ため、使う場合はリソースキャパシティに注意。
