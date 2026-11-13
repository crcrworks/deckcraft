# UPSTREAM — この fork の上流追従と差分

`crcrworks/deckcraft` は [`storytold/deckcraft`](https://github.com/storytold/deckcraft) の fork。
bace（[`crcrworks/bace`](https://github.com/crcrworks/bace)）が LiveWindow に DeckCraft を
プロセス内で埋め込むために持っている。

目的は**改造を bace 側の資産にしないこと**。上流の取り込みを作り直しにしないための規則を
ここに置く。決定の全文と却下した選択肢は bace 側の
[`docs/decisions/0031-craft-fork.md`](https://github.com/crcrworks/bace/blob/main/docs/decisions/0031-craft-fork.md)。

## ブランチ

| ブランチ | 役割 | 動かし方 |
| --- | --- | --- |
| `main` | 上流の mirror。追従専用で、ここには改造を積まない | `git fetch upstream && git merge --ff-only upstream/main`、`git push origin main` |
| `bace` | bace 用の改造を積む唯一のブランチ。bace の依存はここを指す | `git fetch upstream && git merge upstream/main`（merge commit で取り込む） |

- `bace` の起点にした上流の rev: `ef9ac9ac7a1f610656144981b53688efdb9ef518`
  (2026-11-13, "Merge pull request #2062 from cloutier/fix/2060-large-adjustment-guides")
- `main` が mirror している上流の rev: 同上（作った時点で追いついている）
- `bace` が最後に取り込んだ上流の rev: 同上

`main` は bace のビルドには使わない。bace が使うのは `bace` だけ。

## 差分の一覧

`bace` が上流と違うファイル。merge のたびに更新する。

| ファイル | 理由 | `// bace:` |
| --- | --- | --- |
| `UPSTREAM.md` | この文書。コードの差分ではない | — |

**2026-11-13 時点でコードの差分はゼロ。** DeckCraft の `ui-egui` は frame 非依存
（`SlideApp::new(Session, Services)` と `logic(&egui::Context)` / `ui(&mut egui::Ui)`）で、
bace が要るもの（バイト列での開閉、journal、文書 id と revision）は上流の公開 API に既にある:

- 開く: `file.openBytes`（`{name, data: base64}`）
- 書く: `file.saveBytes`（`{format: "pptx"} → {data: base64}`）
- 変更の検出: `Session::active()` の `DocState::uid` / `DocState::revision`
- 差分: `Session::journal`（`Vec<(String, Value)>`、`10,000` 件で先頭 `1,000` 件を捨てる）
- 見本: `deckcraft_engine::sample::open_sample`

`// bace:` の数え方:

```bash
git grep -c '// bace:' bace
git diff upstream/main bace --stat -- . ':!UPSTREAM.md'
```

## 追従の手順

```bash
git remote add upstream https://github.com/storytold/deckcraft.git   # 初回だけ
git fetch upstream
git checkout main && git merge --ff-only upstream/main && git push origin main
git checkout bace && git merge upstream/main
```

衝突したら `// bace:` の付いたハンクを上流の新しい形に付け直す（上流のロジックは書き換えない）。
merge のあと bace 側で lock を進める:

```bash
cargo update -p deckcraft-engine -p deckcraft-ui-egui
```

`apps/bace/Cargo.toml` は `branch = "bace"`、正確な rev は `Cargo.lock` にある。fork のクレートが
要求する `[patch.crates-io]`（`egui` 一式・`krilla`・`zip`）は bace のルート manifest に揃える。
