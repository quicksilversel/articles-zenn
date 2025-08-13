---
title: "Node.jsでマルチCPUコアを使ってスケールする方法（+Kubernetes）"
emoji: "🪄"
type: "tech" 
topics: ["Nodejs","Kubernetes"]
published: true
---

## はじめに

ある日、トラフィックがドカっと増えてアプリが落ちました🥹
「CPUは2コアあるし、余裕でしょ」と思っていたのに、どうやら**1コアしか使われてない**様子…

原因は単純で、**Node.jsはデフォルトでマルチコアを使わない**んですよね。
なので今回は、`cluster`モジュールを使って「疑似マルチコア化」してみた話です。

## そもそもJavaScriptってなんでシングルスレッドなの？

この設計は、ブラウザの主流用途である「DOM操作との親和性」と「直感的な非同期処理」を実現するために選ばれたもの。

> JavaScript is single-threaded by nature. There's no paralleling; only concurrency. Asynchronous programming is powered by an event loop, which allows a set of tasks to be queued and polled for completion.

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Language_overview


**1. DOMとUI操作の安全性**

JavaScriptは、ユーザー操作や画面描画、DOM操作などを一つのスレッド（メインスレッド）で扱う構造になっています。複数スレッドで操作してしまうと、描画とスクリプトが干渉してブラウザが固まるリスクがあるため、あえて「1スレッド—一連の実行」という設計は理にかなっている。

**2. イベント駆動とシンプルな非同期設計**

シングルスレッドでありながら効率的に非同期処理を実現するのが、JavaScriptの強み。setTimeoutやDOMイベント、HTTPリクエストなどは、JavaScriptエンジンではなくブラウザのWeb APIで処理され、完了後にコールバックがイベントループ経由でメインスレッドに戻ります。
これにより、ブロッキングなくコードを進行できるのです。

**3. 開発者にとって扱いやすい設計**

スレッドセーフの保証や競合状態の考慮が不要で、常に一貫した変数の値が保証されるのは大きなメリット（例: 複数の非同期処理が同じ変数を更新しても、競合状態になることはない）。こうした思想は、JavaScriptを勉強・運用する上でも非常に扱いやすい特性になっています。

というわけで、**JavaScriptはブラウザ環境ではあえてシングルスレッドで動くように設計されています。**
…とはいえ、サーバーサイドになると話は別。CPUコアが複数あるのに1つしか使わないのはもったいない場面も多いです。そこで、Node.jsでもマルチコアを活用する方法を見ていきます。

## ステップ1：アプリ側でプロセスをforkしよう

Node.jsはシングルスレッドですが、`cluster`を使うと複数プロセスを立ち上げてマルチコアを活用できます。

```ts
import cluster from 'cluster'
import os from 'os'

const defaultProcessCount = os.cpus().length
const clusterSize = process.env.PROCESS_COUNT ?? defaultProcessCount

if (cluster.isPrimary) {
  for (let i = 0; i < clusterSize; i++) cluster.fork()

  cluster.on('exit', (worker) =>
    console.log(`worker ${worker.process.pid} died`)
  )
} else {
  // ここにアプリ（APIやWebサーバー）の起動処理を書く
}
```

ポイントは利用可能なCPU数を取得して、その数だけforkすること。
あとは各プロセスが別々にリクエストをさばいてくれます。

## ステップ2：Kubernetesのリソース設定を忘れずに

せっかく複数プロセスに分けても、PodにCPUが十分割り当てられていなければ意味がありません。

```yaml
resources:
  limits:
    cpu: "2"
```

もちろん、`PROCESS_COUNT`も環境変数としてdeploymentに入れます。

```yaml
spec:
  containers:
      env:
        - name: PROCESS_COUNT
          value: "2"
```

💡 プロセス数は、割り当てられたCPUコア数に合わせるのがポイントです。
もしPodにCPU 1つしかないのに4プロセスも立てたら、みんなでその1コアを奪い合うことになっちゃいます🥲

**ざっくりした全体像**

```
[ cluster.fork プロセス1 ] ⬅ 決定したCPUコアに
[ cluster.fork プロセス2 ]
         ⋮
Kubernetes Pod with 2 CPU
```

## マルチプロセスとマルチスレッドの違い

上記の流れでマルチプロセスにはできますが、マルチスレッドではないのでCPU負荷の高い処理ではJavaやGoほどの性能は出ないのです。

- cluster = マルチプロセス
  - メモリ空間はプロセスごとに別
  - スレッドみたいにメモリを共有しない
- GoやJavaはマルチスレッド
	- メモリ共有可能なので高効率

## まとめ（TL;DR）

-	JavaScriptはあえてシングルスレッドの設計になっている
-	cluster.fork() = 複数Node.jsプロセスを立てる
-	スレッドではないので、メモリ共有は無し（そのため、Javaよりは性能が低い）

「毎回1コアしか使えてない…」って感じるなら、ぜひ試してみてください！
特に「低コスト環境で最大限パフォーマンス出したい」場面にピッタリです。
内容について、ご意見やツッコミもお寄せいただけると嬉しいです🥰
