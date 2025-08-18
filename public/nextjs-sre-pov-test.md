---
title: EKSでNext.jsを安定運用するためのSRE実践ノウハウ
tags:
  - "nextjs"
  - "sre"
  - "kubernetes"
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

こんにちは。元はフロントエンドエンジニアとしてNext.jsアプリを開発していて、現在はSREとしてそのアプリをEKS上で運用しているゾイです！
Next.jsは「作って動かす」までは簡単ですが、**本番で安定して動かし続ける**にはSRE的な設計・監視・最適化が欠かせません。
この記事では、私が実運用で直面し改善してきたポイントを**EKS+Next.js**に絞って解説します。

---


## 1. ヘルスチェック設計（readiness/liveness/startup）

`readiness`と`liveness`を分けるのは必須です。
外部依存（DB/外部API）がある場合は`readiness`で疎通確認を行い、失敗時はトラフィックを切る。一方、**`liveness`は「プロセス生存確認」**に留め、頻繁に落とさない設定にします。起動が重いSSRは`startupProbe`を使うと安定します。

### ハンドラー例（依存先あり）

```ts
// /api/health/readiness
app.get('/health/readiness/', (_, reply) => {
  const ok = await checkExternalDeps(); // DBや外部APIの疎通確認
  reply.code(200).send({ status: 'READY' })
})
```

### ハンドラー例（依存先なし）

一方で、依存先がないシンプルなアプリなら共通化もOK。

```ts
app.get(/^\/health\/(liveness|readiness)\/?$/, (_, reply) => {
  reply.code(200).send({ status: 'READY' })
})
```

### Probe設定例（EKS）

```yaml
readinessProbe:
  httpGet: { path: /health/readiness/, port: 3000 }
  initialDelaySeconds: 3
  periodSeconds: 10
livenessProbe:
  httpGet: { path: /health/liveness/, port: 3000 }
  initialDelaySeconds: 10
  periodSeconds: 30
startupProbe: # 起動が重いSSRは導入を検討
  httpGet: { path: /health/liveness/, port: 3000 }
  failureThreshold: 30
  periodSeconds: 5
```

**優先度の目安**

- `startupProbe`：起動完了まで`liveness`の実行を遅らせる
- `readiness`：依存先の状態に追随（失敗中はLBから外す）
- `liveness`：プロセスがハングしたら再起動（頻度高すぎると逆効果）


## 2. SSRとスケーリングの誤解

**CPUコアを増やせば比例して速くなるとは限りません。**
Node.jsは基本シングルスレッドで動くため、2コア割り当てても1コアしか使わない状況が起こりえます。`cluster`やプロセスマネージャー（例: PM2）で**プロセスをコア数に合わせて分岐**させるのが定石です。

詳しくは、こちらの記事をどうぞ！
https://zenn.dev/quicksilversel/articles/enable-multi-cpu-nodejs

## 3. SSR/CSRを踏まえた障害試験シナリオ

Next.jsは**SSRとCSRで障害時の見え方が違う**ため、バックエンドの「200/500だけ見る」試験では不十分です。

- **SSR経路**：依存が落ちるとHTTPエラーが返る傾向
- **CSR経路**：画面は200でも**部分的に欠落**（フェイルソフト）しがち

私が組むテストマトリクス（例）

| 観点         | シナリオ       | 期待挙動（SSR）                     | 期待挙動（CSR）                                       |
|--------------|---------------|--------------------------------------|-------------------------------------------------------|
| 依存API障害  | API 5xx       | ページ5xx or エラーページ            | ページは200、該当ウィジェット非表示/プレースホルダー |
| レイテンシ    | API 3s→10s    | SSRタイムアウト/リトライ             | UIローディング表示→タイムアウト扱い                  |
| 認証失効      | セッション切れ | リダイレクト/エラーページ            | クライアントで再ログイン誘導                          |

> 実運用では**画面要素単位の可用性（部分劣化）**を受け入れる設計が多いです。PlaywrightなどのE2Eに**視覚リグレッション**や**特定コンポーネントの可視性チェック**を入れると再現性が高まります。

## 4. キャッシュ設計（CDN×SSR）

CDNを前段に置くだけで、SSRサーバーの負荷は大きく下げられます。重要なのは**なにを・どれくらい**キャッシュするか。

- **静的アセット**（`/_next/static/*`、画像、フォント）
  - 長寿命&ハッシュ付きファイル名 → `Cache-Control: public, max-age=31536000, immutable`
  - 変更時はbuildIdが変更されてファイル名も変わるため追加の破棄は不要（いわゆるcache busting）。

- **SSR HTML**  
  - パーソナライズや認証が絡む場合は**基本ノーキャッシュ**（`Cache-Control: no-store`）。
  - 共有キャッシュ（CDN）のみ短命で許可したい場合は`s-maxage`を短く設定（例: 数十秒）。

**ざっくりした図**

```text
Client → CDN(Akamai/CloudFront等) → ALB → Next.js SSR
                      ↑
            ここにキャッシュを入れる
```

## 5. ログ設計（構造化・相関ID・レイテンシ）

Next.jsはデフォルトではアクセスログが弱いので、カスタムサーバーで**構造化ログ**を出すのが実践的です。Fastifyの`onResponse`で**レイテンシ**と**相関ID**を記録するだけでも、トラブル時の切り分けが劇的に楽になります。


Fastify例（機微情報なし/最小構成）

```ts
import Fastify from "fastify";

const app = Fastify({ logger: true });

app.addHook("onResponse", (req, reply, done) => {
  req.log.info({
    http_method: req.method,
    http_status: reply.statusCode,
    user_agent: req.headers["user-agent"] ?? null,
    latency: reply.getResponseTime() / 1000
  });

  done();
});
```

https://fastify.dev/docs/latest/Reference/Hooks/


**計測したい基本メトリクス**

- レイテンシ（p50/p90/p99）
- エラー率（HTTP 5xx/4xx）
- リクエスト数（リソース別/ルート別）
- 相関ID（分散トレースの起点）

## まとめ（TL;DR）

1. `readiness` / `liveness` / `startup`を正しく設計し、依存障害を巻き込まない
2. `cluster` / PM2で**1Podの並列度**を上げ、必要に応じてHPAで**水平スケール**する
3. SSR/CSRの違いを踏まえた障害試験（部分劣化を含めて検証）
4. CDNキャッシュ方針を明確化（静的は長寿命、SSR HTMLは慎重に）
5. 構造化ログ+相関IDで可観測性を底上げ

Next.jsの運用に困っていたら、ぜひ試してみてください！
内容について、ご意見やツッコミもお寄せいただけると嬉しいです🥰
