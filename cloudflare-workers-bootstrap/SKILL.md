---
name: cloudflare-workers-bootstrap
description: Cloudflare Workersで開発を始めるときのbootstrap prompt。新規プロジェクトの雛形作成に必要な前提確認・wrangler設定・環境分離・デプロイ導線・観測手段を、コピペ可能なpromptテンプレートとして切り出したもの。「Workersではじめて」「wranglerの初期設定をして」「Cloudflareにデプロイする構成を作って」のように新規立ち上げを指示された時に使う。
allowed-agents: claude-code
---

# cloudflare-workers-bootstrap

Cloudflare Workersで開発を始めるときのbootstrap prompt。Workers特有の前提（エッジランタイム・バインディング・環境分離）を最初に固めないと、後からの作り直しになる。本skillはその前提集めと雛形作成を1つのpromptに束ねたもので、プロジェクト固有の事情は `{{変数}}` で埋める。

wranglerのコマンド体系・設定項目は流れが速い。正確なフラグやバインディング形状は訓練知識に頼らず、その時点で最新のドキュメントで裏を取る（https://developers.cloudflare.com/workers/wrangler/）。

## When to use

- 「Cloudflare Workersで〇〇を作って」のように新規プロジェクトの立ち上げを指示された時
- 既存プロジェクトへWorkersデプロイの導線を足す時（テンプレートの該当節だけ使う）
- wrangler.jsonc / 環境分離 / デプロイ手順の雛形が欲しい時

既存Workersプロジェクトの日々の運用（デプロイ・ログ・シークレット操作）だけなら本skillは要らない。wranglerの操作リファレンスが必要な場合は `wrangler` skillを使う。

## Bootstrap prompt テンプレート

以下をそのまま貼り、`{{変数}}` を埋めて使う。聞き返しが必要な項目は消さず、利用者に質問してから雛形を作る。

```markdown
Cloudflare Workersで {{作りたいもの1行}} を始めるための雛形を作ってください。

前提:
- Worker名: {{worker-name}}
- ランタイム: {{エッジのみ / Node.js互換が必要か}}
- 使うバインディング: {{例: D1(読み書き) / KV(キャッシュ) / R2(画像保存) / Workers AI / Queues / なし}}
- 環境: {{production のみ / production + staging}}
- シークレット: {{例: 外部APIキー1件 / なし}}
- デプロイ方法: {{wrangler deploy直実行 / CIから}}

作るもの:
1. wrangler.jsonc（最小構成＋必要なバインディング。compatibility_dateは直近、
   Node.js互換が要るならcompatibility_flagsにnodejs_compat）
2. エントリポイント（fetchハンドラの最小形＋型生成 `wrangler types` の置き場所）
3. .dev.vars.example（ローカル用シークレットの雛形。本文書なし・値は空）
4. ローカル開発・デプロイ・ロールバックの手順（コマンド列で。以下を必ず含める）
   - bun run dev相当: `wrangler dev`（必要なら --env）
   - デプロイ前検証: `wrangler deploy --dry-run`
   - ロールバック: `wrangler versions list` / `wrangler rollback`
   - 観測: `wrangler tail`（--status error / --search）
5. 落とし穴の確認（下の「新規立ち上げのチェックリスト」を1件ずつ潰すこと）

制約:
- シークレットの値はコード・設定・チャットに書かない（`wrangler secret put` で対話入力）
- 推測で進めず、不明なフラグ・設定項目はその場でドキュメントを確認する
```

## 新規立ち上げのチェックリスト

雛形を作ったら、下記を1件ずつ潰す。どれも後から気付くと作り直しになる。

- [ ] **ランタイム**: Node.js API（fs・net・タイマー等）を使うなら `nodejs_compat` を最初から付ける。後付けは動いていた部分まで壊す
- [ ] **compatibility_date**: 直近の日付にする。古いままだと新機能が使えず、上げるたびに挙動が変わる
- [ ] **バインディング名**: コードが読む名前と wrangler.jsonc の binding を完全一致させる。ズレは実行時まで検出できないので `wrangler types` をCIに入れる
- [ ] **環境分離**: staging が要るなら最初に `env.staging` を切る。後付けは本番リソースと名前が衝突する
- [ ] **ローカルと本番の差**: ローカルのバインディングは既定でシミュレーション。AIなど remote 必須のものは `remote: true` を明示し、課金が発生することを伝える
- [ ] **シークレットの置き場所**: ローカルは `.dev.vars`（コミットしない）、本番は `wrangler secret put`。`vars` に秘密を書かない
- [ ] **永続データの置き場所**: D1・R2・KV のどれに何を置くかを決める。D1マイグレーションは適用順が命なので、番号付きファイルで管理する
- [ ] **デプロイの戻し方**: `versions list` / `rollback` が使えることを確認する（壊れたデプロイを直す唯一の速い道）
- [ ] **ログの見方**: `wrangler tail` が届くことを確認する（`--status error` / `--search` の絞り込み方と一緒に伝える）

## 最小構成の例

wrangler.jsonc（詳細なバインディング形状は `wrangler` skill 参照）:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "{{worker-name}}",
  "main": "src/index.ts",
  "compatibility_date": "{{直近の日付}}",
  "vars": {
    "ENVIRONMENT": "production"
  },
  "env": {
    "staging": {
      "name": "{{worker-name}}-staging",
      "vars": { "ENVIRONMENT": "staging" }
    }
  }
}
```

エントリポイント（`src/index.ts`）:

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname === "/api/health") {
      return Response.json({ ok: true, env: env.ENVIRONMENT });
    }
    return new Response("Not found", { status: 404 });
  },
} satisfies ExportedHandler<Env>;
```

`.dev.vars.example`:

```
# ローカル開発用のシークレット。コピーして .dev.vars を作る（コミットしない）
# API_KEY=
```

## 使い方ドキュメント（運用側）

立ち上げ後の日々の運用は下記の順で回す。初回は雛形と一緒に伝える。

1. `wrangler dev` でローカル起動（バインディングは既定でシミュレーション）
2. `wrangler types` で型を更新（バインディングを変えたら必ず）
3. `wrangler deploy --dry-run` で検証してから `wrangler deploy`
4. 壊したら `wrangler rollback`（`versions list` で版を確認）
5. 調査は `wrangler tail --status error` から（必要なら `--search` で絞る）

## やってはいけないこと

- **シークレットを平文で残さない。** コード・wrangler.jsonc・チャットログ・スクリーンショットに出さない
- **本番リソースで試さない。** D1・R2・KV の作成・削除の試行は staging かローカルでやる
- **古い知識で断定しない。** wrangler のフラグ・設定項目は変わりやすいので、使うたびにドキュメントで確認する
- **プロジェクト固有の事情をテンプレートに焼き込まない。** 変わる部分は `{{変数}}` のまま残し、埋めるのは使う側の仕事にする
