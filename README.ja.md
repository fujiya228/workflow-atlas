# Workflow Atlas

*English: [README.md](README.md)*

**プロジェクトの状態・操作・フローを JSON で仕様化し、1枚のインタラクティブ HTML 仕様書として読む。**
図を「描く」のではなく、仕様を「歩く」。

*node* は **状態**(テーブル / ストア / コンポーネント / アクター — システムが取り得る・保持するもの)。
*flow* は **操作**(状態を変換する順序つきステップ列)。書くのは JSON だけ。汎用エンジンが、キーボードで
歩ける・9 種のカラースキームを切替できる、外部依存ゼロの単一 `.html` を描画します。

[![Workflow Atlas デモ — 操作を選び、方向キーでステップを送る(キャンバスと右パネルが同期)、最後にカラースキーム切替](docs/media/demo.gif)](https://fujiya228.github.io/workflow-atlas/)

**▶ ライブギャラリー:** https://fujiya228.github.io/workflow-atlas/

## なぜ

手描き / mermaid / graphviz の図は「何と何が繋がるか」には答えますが、システムが変わると腐ります。
Workflow Atlas は「どの操作が状態 A から B へ動かすのか、ステップはどの順で走るのか」に答え、しかも
単一の編集可能な JSON ドキュメントのまま保てます。

## インストール

**Clone + symlink**(Claude Code スキルとして使う):

```bash
git clone https://github.com/fujiya228/workflow-atlas ~/code/workflow-atlas
ln -s ~/code/workflow-atlas/skills/workflow-atlas ~/.claude/skills/workflow-atlas
```

**プラグインマーケットプレイス**(Claude Code):

```
/plugin marketplace add fujiya228/workflow-atlas
/plugin install workflow-atlas@workflow-atlas
```

実行時依存はありません。ビルドスクリプトは Node(stdlib のみ)で動き、インライン方式は何も要りません。

## 使い方

```bash
cd skills/workflow-atlas

# 同梱例をビルド (Tier A — Node 必要・Windows 含む全OS共通)
node scripts/atlas build assets/examples/rag-agent.ja.json --theme okabe-ito-light -o /tmp/atlas.html

# 自分の atlas を作る
cp assets/examples/rag-agent.ja.json my.json   # my.json を自分のシステムに合わせて編集
node scripts/atlas build my.json --theme dracula -o my.html
```

**Node が無い / Windows / AI が代行する場合(Tier B)**: `assets/template.html` をコピーして、埋め込みの
`#workflow-data` JSON ブロックと `data-theme` 属性を直接編集するだけ。ツール不要・全OS。スキルとして
呼ばれた場合は **この編集を Claude 自身が行います**(
[`references/build.md`](skills/workflow-atlas/references/build.md))。

HTML を開いたら:

- **数字キー `1`–`0`** で操作(フロー)を選択
- **方向キー `← →` / `↑ ↓`** でステップを送る・戻す(`Home`/`End` で先頭・末尾)
- **テーマのドロップダウン**で 9 種のカラースキームを切替
- **ホイール/ドラッグ**でズーム・パン、**`F`** で全体表示、**`Esc`** で解除

## カラースキーム

9 プリセット、既定は `okabe-ito-light`(色覚多様性に配慮した可視化パレット — ターミナル見た目より
判読性を優先): Okabe-Ito(light/dark)、Catppuccin(Latte/Mocha)、Solarized(light/dark)、Nord、
Dracula、Gruvbox Dark。詳細は
[`references/themes.md`](skills/workflow-atlas/references/themes.md)。

## ドキュメント

- [`SPEC.md`](SPEC.md) — 規範仕様(JSON スキーマ・テーマ・描画・キー操作・ビルド・適合性)
- [`skills/workflow-atlas/SKILL.md`](skills/workflow-atlas/SKILL.md) — スキル本体(エントリポイント)
- `skills/workflow-atlas/references/` — schema / authoring / themes / keybindings / build

## ライセンス

MIT — [LICENSE](LICENSE) を参照。
