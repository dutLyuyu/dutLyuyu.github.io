# Git ワークフロー障害報告書

---

## 1. 概要

feature ブランチ（prep-trivy）の diverge 解決作業中に、冲突解決 commit が GitHub auto-merge を
連動させ、意図しない revert が公共ブランチに混入した。
本報告では事故の経緯、根本原因、我々のワークフロー設計の背景、
他ワークフローとの比較、および再発防止策を記録する。

---

## 2. 用語定義

| 用語 | 説明 |
|---|---|
| feature ブランチ (prep-trivy) | 複数人が共有する公共ブランチ。Decentralized but centralized。 |
| 個人ブランチ | 各開発者が feature ブランチから切った作業ブランチ |
| diverge | main と feature ブランチの履歴が乖離した状態 |
| conflict-resolve | diverge 解決のために main HEAD から一時的に切るブランチ |
| S1 | PR2932 の冲突解決 commit（三方マージ commit） |

---

## 3. 事故経緯

### 3.1 ブランチ初期状態

```
main:            common ─ A ─ B ─ C   (新 commit が追加され diverge 発生)
feature(prep-trivy): common ─ P1      (feature 独自 commit あり)
conflict-resolve:    common ─ A ─ B ─ C  (main HEAD から切出し、追加 commit なし)
```

PR の状態：
- **PR2932**: prep-trivy (head) → conflict-resolve (base)　※ diverge 解決 PR
- **PR2926**: conflict-resolve (head) → prep-trivy (base)　※ auto-merge 有効

### 3.2 事故タイムライン

```
Step 1  PR2932 の冲突を手動解決
        → three-way merge commit S1 を prep-trivy に push
          Base: common, Ours: P1, Theirs: C1(conflict-resolve HEAD)
          S1 の parents = { P1, C1 }

Step 2  S1 push により C1 が S1 の ancestor になる
        → PR2926（conflict-resolve → prep-trivy）が fast-forward 可能な状態に

Step 3  PR2926 の auto-merge が発動
        → fast-forward merge のため新 commit を生成せず
        → merge commit = prep-trivy HEAD = S1
        → S1 が「PR2932 冲突解決 commit」と「PR2926 merge commit」の両方に帰属

Step 4  異常を検知。慌てて S1 を revert
        → revert commit fcdb13b を prep-trivy に push
        → conflict-resolve ブランチを削除（main へ未 merge のまま）

Step 5  PR2934（prep-trivy → main）を merge
        → fcdb13b（S1 の revert）が main に混入
        → PR2926 でマージされた内容が main 上で抹消される
```

### 3.3 事故の構造図

```
common ──── P1 ──── S1 ──── fcdb13b (revert S1)
             \     / \                  \
              \   /   \                  └── main へ混入（PR2934）
               \ /     └── PR2926 merge commit（同一 SHA）
                C1
                ↑
        conflict-resolve HEAD
```

---

## 4. 根本原因分析

### 4.1 技術的根本原因：three-way merge commit の構造的必然

冲突解決 commit S1 は **merge commit** であり、その parents に C1（conflict-resolve HEAD）が含まれる。

```
S1.parents = { P1, C1 }
→ C1 is ancestor of S1 ✅
→ PR2926（C1 → prep-trivy）は fast-forward merge 可能
→ fast-forward では新 commit を生成しない
→ PR2926 の merge commit = S1（prep-trivy HEAD）
```

これはタイミングの偶然ではなく、**three-way merge commit を push した瞬間に
逆方向 PR の fast-forward 条件が構造的に成立する**という必然である。

### 4.2 プロセス上の原因

| # | 原因 | 詳細 |
|---|---|---|
| P1 | auto-merge の影響範囲の未把握 | PR2926 に auto-merge が設定されていたが、冲突解決操作が merge 条件を満たすことを認識していなかった |
| P2 | PR 間の依存関係の未整理 | PR2932 と PR2926 が逆方向の loop 関係にあることを事前に確認していなかった |
| P3 | 慌てた revert の影響範囲の未確認 | S1 が PR2926 の merge commit でもあるため、revert により PR2926 の内容が失われることを確認せずに revert を実行した |
| P4 | SOP の不備 | conflict-resolve ブランチを使った diverge 解決の手順が文書化されておらず、操作判断が個人に依存していた |

---

## 5. 我々のワークフロー設計

### 5.1 feature ブランチの位置づけ

```
        main（保護）
          │
   ┌──────┴──────┐
feature (prep-trivy)    ← 複数人共有、Decentralized but centralized
   ├── 個人ブランチA
   ├── 個人ブランチB
   └── 個人ブランチC
```

複数人が同一の大きな機能に取り組む際、main とは独立した共有ブランチを設ける。
個人ブランチは feature から切り出し、feature への PR で統合する。

### 5.2 feature ブランチで rebase できない理由

```bash
# ❌ 禁止操作
git rebase main  # feature ブランチ上で実行
```

feature ブランチは複数人が共有しているため、履歴の rewrite を行うと
すべての個人ブランチの親が無効になる。force push は運用上禁止。

### 5.3 diverge 解決方針

冲突の範囲とブランチの状態に応じて 2 つのケースに分類する。

---

#### Case 1：冲突範囲が局所的（package.json 等のバージョン管理ファイルのみ）

**適用条件（両方を満たすこと）：**

```
条件A: 変更がまだ公共ブランチに push されていない
       OR
       すでに push 済みだが、公共ブランチが全員周知の code freeze 状態にある

条件B: 冲突範囲が package.json 等の局所的なファイルのみ
```

**例：** feature ブランチでの開発が完了し、直近でリリースバージョン番号を
feature に push した。他は main と差異がなく、package.json だけが異なる。

**解決手順（最もよく使われる方法）：**

```bash
# 1. 変更を一時退避
git revert <version-bump-commit>   # または
git stash

# 2. 最新コードを取得
git pull

# 3. 変更を再適用
git stash pop
# → 再度バージョン番号を更新して commit
```

操作対象は**個人ブランチ・公共ブランチのどちらでも可**。
ただし公共ブランチで操作する場合は code freeze が全員に周知されていること。

---

#### Case 2：並行する長期ブランチ間の冲突（chore/A ∥ feature/B）

リリースタイミングが異なる 2 つのブランチが並行している場合。

**例：** `chore/A`（未リリース・長期継続）と `feature/B` が並走し、冲突が発生している。

**冲突あり：中間ブランチ方式**

```
先にリリースされるブランチ（例: chore/A）から中間ブランチを切る：

  git checkout chore/A
  git checkout -b feature/A-B-merge

PR を作成:
  後リリースブランチ(feature/B)  →  feature/A-B-merge
  冲突修正は後リリースブランチ（feature/B）上に落ちる

完了後:
  feature/A-B-merge は close + 削除（merge 不要）
```

```
chore/A ──────────────────────── (先 release)
    │
    └── feature/A-B-merge  ← base になるだけ
                 ↑
feature/B ───── S1  (冲突修正 commit が feature/B 上に落ちる)
```

**冲突なし：直接 PR**

```
先リリースブランチ → feature/B  の PR を直接作成（中間ブランチ不要）
```

---

#### PR 作成前の反向 PR チェック（必須）

> **本事故の直接的な予防点**

PR を作成する前に、**逆方向の open PR が存在しないか必ず確認する。**
存在する場合は**先に close してから**新しい PR を作成する。
警告だけでは不十分であり、close が必須。

```
例：
  これから作成する PR: feature/B → feature/A-B-merge
  確認対象: feature/A-B-merge → feature/B  の open PR が存在しないか

  → 存在する場合: 先に close → その後新 PR を作成
```

auto-merge が有効な状態で反方向 PR が open のまま冲突解決 commit を push すると、
fast-forward により意図しない merge が自動で発動する（本事故の根本原因）。

### 5.4 公共ブランチ上での禁止操作

feature ブランチは複数人が共有するため、以下の操作は**いかなる場合も禁止**する。

```bash
# ❌ 禁止: 公共ブランチ上での rebase
git rebase main

# ❌ 禁止: 公共ブランチ上での cherry-pick
git cherry-pick <commit>
```

どちらも**履歴の rewrite または予期しない commit の挿入**を引き起こし、
全員の個人ブランチとの整合性が崩れる。

| 操作 | 問題 |
|---|---|
| `git rebase` | commit SHA が変わり、全員の個人ブランチの親が無効になる。force push が必要になり共有ブランチが破壊される |
| `git cherry-pick` | 同一内容の commit が重複して発生し、後続の merge・rebase 時に予期しない冲突を引き起こす |

diverge の解消は必ず §5.3 の Case 1 / Case 2 の手順に従い、
merge commit として取り込む方法で行う。

---

## 6. 他ワークフローとの比較

### 6.1 比較表

| 項目 | Git Flow | GitHub Flow | 我々のフロー |
|---|---|---|---|
| 長期公共ブランチ | develop | なし | feature (prep-trivy) |
| 個人ブランチの base | develop | main | feature |
| diverge 解決方法 | develop ← main merge | PR + rebase | conflict-resolve 中間ブランチ |
| main 保護 | あり | あり | あり |
| リリース管理 | release ブランチ | tag | main merge |
| 複数人同一機能 | develop で吸収 | 短命ブランチ推奨 | feature で共有 |

### 6.2 Git Flow との違い

Git Flow は develop ブランチが常に存在し、main との diverge は
`git merge main → develop` で定期的に吸収する。
しかし Git Flow は develop へのアクセス権を全員に与えるため、
保護ルールが強い環境（外部コントリビュータ混在など）では運用が難しい。

### 6.3 GitHub Flow との違い

GitHub Flow はブランチを短命に保つことで diverge 自体を発生させない。
しかし複数人が長期間同一機能を開発する場合、
全員が main から直接 PR を出すと統合コストが高くなる。
共有 feature ブランチを一枚挟むことでレビューの粒度を分離している。

### 6.4 我々のフローを選んだ理由

- main の保護ルールを維持しつつ
- 複数人が共有コンテキスト上で作業するために
- feature ブランチを "mini-main" として機能させている

trade-off として、**feature ブランチと main の diverge 管理が手動**になるため、
SOP と自動検出の仕組みが必要になる。

---

## 7. 教訓と再発防止策

### 7.1 技術的予防策

**① diverge 解決中は関連 PR の auto-merge を無効化する**

```
conflict-resolve ブランチを切る前に：
- PR2926（conflict-resolve → feature）の auto-merge を OFF にする
- 全作業完了後に必要に応じて再設定する
```

**② PR 作成前の反方向 PR チェックを SOP に組み込む**

PR を作成する前に反方向の open PR が存在しないかを確認し、
存在する場合は**先に close してから**新しい PR を作成する。
警告だけでは不十分。close が完了するまで新 PR の作成を保留する。

GitHub Actions による自動コメントは補助的な安全網として有効だが、
あくまで人的チェックの補助であり、自動検出を信頼して手順を省略しない。

**③ revert 実行前の影響範囲確認チェックリスト**

```
revert 前に確認すること：
□ 対象 commit は他の PR の merge commit でもあるか？
  → git log --merges --all | grep <SHA>
□ 対象 commit が merge commit の場合、-m 1 オプションが必要か？
□ revert 後、該当 PR の内容が失われないか？
```

### 7.2 プロセス的予防策

**④ conflict-resolve 作業中の team lock**

```
作業開始時：チャンネルに通知 🔒 "conflict-resolve 作業中。feature への push 禁止"
作業完了時：チャンネルに通知 ✅ "作業完了。git pull --rebase してください"
```

feature ブランチへの push が並走すると `--onto` の anchor がずれる。

**⑤ diverge 解決 SOP の文書化**

本報告書をベースに SOP を整備し、口頭伝承に依存しない体制を作る。

### 7.3 発動条件の明確化

| 状態 | 対応 |
|---|---|
| feature に独自 commit なし、main が進んでいる | 全員 rebase（§5.4 の条件） |
| feature に独自 commit あり | conflict-resolve ブランチで diverge 解決 |
| 個人ブランチが多数かつ大規模 | 段階的 rebase。リードが調整 |

---

## 8. 修復方案と検証

### 8.1 今回の修復

1. `fcdb13b`（revert commit）を再 revert する

```bash
git checkout main
git revert fcdb13b
# → PR2926 の内容が復元される
```

2. main の diff を確認し、期待する状態と一致するか検証する

```bash
git diff <expected-state> main
```

### 8.2 証拠チェーンの検証スクリプト

事故の証拠チェーンを検証するために 2 つのスクリプトを作成した。

**verify_incident.sh** — 実際のリポジトリで 4 命題を検証する

```
命题1: incident commit（S1）がリポジトリに存在するか
命题2: revert commit の parent が S1 と一致するか
命题3: revert commit が TARGET_BRANCH に混入したか（オプション）
命题4: S1 の内容が TARGET_BRANCH 上で失われているか（オプション）
```

```bash
# 基本実行（命題1・2のみ）
bash verify_incident.sh <S1_SHA> <REVERT_SHA> main

# 命題3・4も含めて検証する場合
CHECK_MAIN=true bash verify_incident.sh <S1_SHA> <REVERT_SHA> main
```

今回の実行結果：

```
── 命题1: incident commit 实际存在 ─────────────────
  ✅ PASS  commit object exists in repo
           commit msg : Merge branch 'conflict-resolve-test' into prep-trivy

── 命题2: revert commit 指向 incident commit ───────
  ✅ PASS  revert msg contains 'Revert'
  ✅ PASS  revert parent === incident commit
           revert msg : Revert "Merge branch 'conflict-resolve-test' into prep-trivy"

════════════════════════════════════════════════════
  ✅ ALL 3 checks passed — 事故证据链（命题1・2）完整
════════════════════════════════════════════════════
```

命題3・4 は FAIL が正常（人為介入により main 混入を阻止したため）。

**sandbox_incident.sh** — ローカルで事故シナリオを再現し構造を検証する

```bash
bash sandbox_incident.sh
# /tmp 以下に一時リポジトリを作成し verify_incident.sh を自動実行
```

fast-forward が成立する構造的根拠の確認：

```bash
# S1 の親に C1 が含まれる = PR2926 の fast-forward 条件が成立する
git merge-base --is-ancestor $C1 $S1 && echo "✅ C1 is ancestor of S1"
```

---

## 9. まとめ

今回の事故は、diverge 解決のための **three-way merge commit（S1）が
逆方向 PR（PR2926）の fast-forward 条件を構造的に成立させる**という、
git の merge commit の性質に起因するものであった。

タイミングの問題でも、auto-merge の設定ミスだけでもなく、
**この操作順序では必ず同じことが起きる**という点が重要である。

再発防止の核心は：

> conflict-resolve ブランチを切る前に、
> 逆方向 PR の auto-merge を必ず無効化する

これを SOP として明文化し、チェックリストに組み込むことが最優先の対策である。