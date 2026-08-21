# GitHub Project セットアップ手順（管理者向け・1回だけ）

複数リポジトリで並行作業するエージェントの状況を、人間が1画面で見るためのProjectを用意する。claimプロトコル本体は [SKILL.md](SKILL.md)。

**前提: エージェントはProjectに書き込まない。** Projectは Issue/PR の状態から automation で自動投入される投影であり、claimの権威ではない（理由は SKILL.md「なぜclaimをGitHub Projectに置かないか」）。

## 設計上の制約

エージェントがProject APIを呼ばないので、**Projectのfieldは Issue/PR の状態から自動導出できるものに限る**。`Executor` や `Last Heartbeat` のように導出できないものをボードに出したい場合は、PR descriptionから同期するActionsを別途置く（手順6、任意）。

## 1. Projectを作る

owner配下に1つだけ作る（例: `AI Work`）。組織ならorganization project、個人ならuser project。どちらも**複数リポジトリのIssue/PRを1つのProjectに集められる**。

Issue自体は原則として変更対象のリポジトリに作る。Projectは横断ビューを提供するだけで、Issueの置き場所ではない。

## 2. Status のオプションを定義する

`Claiming` / `In Progress` / `Review` / `Blocked` / `Done` / `Abandoned`

SKILL.md のclaimメタデータの `status` と同じ語彙にする。ボードの列と、エージェントがPR本文に書く文字列が一致するので、人間が突き合わせずに読める。

Issueのopen/closeとProjectのStatusは別物として扱う:

| Issue | Project Status |
|---|---|
| open | Claiming / In Progress / Review / Blocked |
| closed (`completed`) | Done |
| closed (`not_planned`) | Abandoned |
| closed (`duplicate`) | Projectから外す |

## 3. auto-add workflow を設定する

Project の Workflows → **Auto-add to project**。フィルタ例:

```
is:issue is:open label:in-progress
```

対象リポジトリごとに1つずつ設定が要る（リポジトリを増やしたら追加する）。これによって、エージェントが `CLAIM_LABEL` を付けるだけでProjectにitemが乗る。

## 4. built-in workflow を有効にする

- **Item closed** → Status: `Done`
- **Pull request merged** → Status: `Done`
- **Item added to project** → Status: `In Progress`（任意）

`Closes #N` を書いたPRがmergeされると Issue が自動closeされ、この workflow が Done にする。エージェントは何もしない。

## 5. View を作る

| View | 設定 | 用途 |
|---|---|---|
| Active | Board。Status: Claiming / In Progress / Review / Blocked | 今何が動いているか |
| By repository | Group by: Repository、Filter: `is:open` | どのリポにエージェントが集中しているか |
| Review | Status = Review | AIが仕事を終え、人間/CI待ち。**最も見る価値がある** |
| Blocked | Status = Blocked | 人間の判断が要るもの |

Repository は Project が最初から持っているので、custom field を作る必要はない。

## 6. （任意）claimメタデータの同期

`Executor` / `Last Heartbeat` / `Resource` をボードに出したい場合のみ。PR description の `## Claim` ブロックをパースして `gh project item-edit` で書くActionsを、対象リポの `pull_request` イベントに仕掛ける。

- **Actions の `GITHUB_TOKEN` では Projects v2 を操作できない。** `project` scope を持つPAT、またはprojects権限を付けたGitHub Appのinstallation tokenが要る
- ローカルから `gh` で操作する場合も、`project` scope はgh既定のスコープに含まれない（`gh auth refresh -s project` が必要）
- **無くてもプロトコルは完全に動く。** ボードの情報量が増えるだけなので、token管理のコストと釣り合うときだけ入れる

## 7. 容量に注意する

Projectは active + archived 合わせて **50,000 item** が上限。1 Issue = 1 work item を守っていれば到達しないが、agent run / tool invocation / retry ごとにIssueを作り始めると数日で埋まる。

- Project / Issue = **work item**
- 外部DB / OpenTelemetry / エージェント基盤 = **execution log**

と分ける。SKILL.md の「retryで新しいIssueを作らない」はこの制約と対になっている。

## 注意

- Projectの表示は automation 経由なので**遅延する**。エージェントのclaim判定に使ってはいけない
- Project の field は last-write-wins で、複数セッションが同じitemを触ると壊れる。権威ではないので許容範囲だが、そこに真実があると思ってはいけない
