# figma-contrast-check-lite — Figmaフレームの WCAG コントラストチェック（軽量版）

FigmaのFRAME / SECTION / COMPONENT / COMPONENT_SETを指定すると、テキストの実効的な前景色・背景色（覆う兄弟レイヤーや祖先のFillを合成して解決）を求め、WCAG AA基準のコントラスト比をチェックするClaude Codeスキルです。

これは有料講座で提供されているフル版`figma-contrast-check`（AA/AAAの選択、非テキストUI要素の参考チェック、確認後の自動修正機能つき）から、それらの機能を取り除いたトリムダウン版です。判定基準はWCAG AA固定、対象はテキストのみ、そしてFigmaファイルへの書き込みは一切行わない「チェック＆レポートのみ」の軽量版です。**コントラストチェックの中身がどんなものか、まず試してみたい方**に向いています。

> **フル版が気になる方へ**: この軽量版はWCAGコントラストチェックのコアロジック単体を体験いただくものです。フル版ではAAA判定・非テキストUI要素（ボタン枠線・アイコンなど）の参考チェック・確認ゲート付きの自動修正が加わります。他のFigma連携スキル（`figma-audit`、`figma-layer-rename`、`figma-tokenize`など）と組み合わせて使うことも想定しています。全スキルカタログはこちら: [KMRVID Claude Skills README](https://fragrant-edam-563.notion.site/KMRVID-Claude-Skills-README-md-3935dae25dd280e295b6cafc1d47d0f6)

> **有料講座のプレビュー**: 講座内の各種スキルを実際に動かしている様子は、こちらの[YouTube](https://www.youtube.com/@kmrvid/videos)や講座の[無料プレビュー](https://kmrvid.com/apps/ldt-course/course/6922c0ad336ff5545bbff61d/6a40a8485dd31d7633eb902e?locale=ja)（アカウント作成、ログイン不要）に掲載しております。

---

## これは何か

色のアクセシビリティは、実装後のコードではなくデザインの時点で決まってしまうことがほとんどです。実装が終わった後でコントラスト不足が見つかると、デザインとコードの間で手戻りが発生します。このスキルはFigmaの色指定そのものに対してWCAGの計算式を直接適用し、実装に進む前に問題を洗い出します。

```
Figma FRAME/SECTION/COMPONENT URL
        │
        ▼
テキストごとに実効的な前景色・背景色を解決
  （覆う兄弟レイヤー → 祖先のFill → アルファ合成 → ページ背景）
        │
        ▼
WCAG AAの比率を計算し、通常/大きい文字を判定して閾値と比較
        │
        ▼
レポート表を出力（Pass / Fail / 手動確認）
```

`INSTANCE`のサブツリーもスキップしません — インスタンス内であっても実際にレンダリングされて読まれるテキストなので、チェック対象から外しません。

---

## スコープ

- **対象:** WCAG 1.4.3（テキストのコントラスト）をAAレベルで判定。判定に使う背景色は、直近の親のFillだけでなく、実際に合成された結果（覆う兄弟レイヤーや祖先のFill、不透明度の重なりを考慮）を使用
- **非対応:** AAA判定、非テキストUI要素（ボタン枠線・アイコンなど）のコントラスト、Figmaファイルへの書き込み・自動修正。これらが必要な場合はフル版`figma-contrast-check`をご利用ください
- `figma-audit`（構造面のAI可読性監査）と組み合わせて使うと、「AIが正確に実装できる状態か」と「実装結果が実際に読める配色になっているか」の両方をチェックできます

---

## リポジトリ構成

```
skills/
  figma-contrast-check-lite/
    SKILL.md          — Claude Codeが読み込むスキル定義
    LICENSE
    references/
      wcag-contrast-algorithm.md   — 輝度・コントラスト比の計算式、背景色解決アルゴリズム
```

---

## はじめかた

**1. このリポジトリをclone、もしくはZIPダウンロードして解凍**

```bash
git clone https://github.com/gaspanik/figma-contrast-check-lite-skill
```

**2. Claude Codeにスキルをインストール**

```bash
cp -r skills/figma-contrast-check-lite ~/.claude/skills/
```

**3. Figma MCPの接続を確認**

このスキルは公式Figma MCPサーバーの`use_figma`を読み取り専用で呼び出します。書き込みは行いません。

**4. スキルを実行**

```
/figma-contrast-check-lite https://www.figma.com/design/<fileKey>/...
```

```
このFigmaフレームの色のコントラストをWCAG基準でチェックして: https://www.figma.com/design/...
```

```
Check this frame's color contrast against WCAG AA: https://www.figma.com/design/...
```

※呼び出し時に日本語を含めることで日本語での応答が可能ですが、スキル本体は英語で書かれているため、稀に英語での応答が返ることがあります。

---

## 実行時の流れ

1. **URLの解析とスコープ確認** — FRAME / SECTION / COMPONENT / COMPONENT_SETかどうかを確認
2. **テキストノードのスキャン** — 文字単位のセグメントごとに前景色・背景色を解決
3. **コントラスト比の計算** — 文字サイズ・太さから「大きい文字」判定を行い、WCAG AAの閾値（通常4.5:1 / 大きい文字3:1）と比較
4. **レポート出力** — レイヤーごとのPass / Fail / 手動確認の一覧表と、合格・不合格件数のサマリー

---

## さらに先へ

この軽量版はコントラストチェックのコアロジック単体を体験できるものです。AAA判定・非テキストUI要素の参考チェック・確認ゲート付き自動修正を含むフル版`figma-contrast-check`、および他のFigma連携スキルを含む全スキルカタログは講座に含まれています: [KMRVID Claude Skills README](https://fragrant-edam-563.notion.site/KMRVID-Claude-Skills-README-md-3935dae25dd280e295b6cafc1d47d0f6)

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
