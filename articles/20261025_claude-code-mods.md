---
title: "Claude Code Modsで作る自分専用のTerminal UI"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claude", "claudecode", "mods", "typescript", "terminal"]
published: true
---

## はじめに

Claude Code [Mods](https://code.claude.com/docs/ja/plugins/mods/overview) を作ってみました!

Modsを作った理由は、**TerminalのUIを自由にカスタマイズ** したかったからです

ModsはTypeScriptで作成しますが、**強力な型定義**のおかげで、Claudeに**自然言語で指示するだけで、正確にエラーなく**カスタマイズできました!!

## 対象者

- Claude Codeを利用している方
- Modsを利用したTerminal UIのカスタマイズ例が知りたい方

## Modsとは

2026/10にリリースされた機能で、利用するにはClaude Code v2.1.287 以降が必要です

Modsの仕様は、標準skill `plugin-authoring` で提供されており、TypeScriptの型定義やコード例が含まれています

詳しくはClaude Codeに **Claude Code Modsについて教えてください** と質問してみてください

- `reference.md`: 概要 (全event、ui.render、$.state)
- `types/`: 型定義 (API)
- `examples/`: コード例

![1.png](/images/20261025_claude-code-mods/1.png)

## 作ったMods

### 1. statusline

今までは、Claude Code [ステータスライン](https://code.claude.com/docs/ja/statusline) 機能を使い、ShellScriptで表示していました

![2.png](/images/20261025_claude-code-mods/2.png)

:::details 旧statusline.sh
```bash
#!/bin/bash
input=$(cat)

MODEL=$(echo "$input" | jq -r '.model.display_name')
DIR=$(echo "$input" | jq -r '.workspace.current_dir')
USED=$(echo "$input" | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
FIVE_H=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // 0' | cut -d. -f1)
FIVE_RESET=$(echo "$input" | jq -r '.rate_limits.five_hour.resets_at // empty | strflocaltime("%H:%M")')
SEVEN_D=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // 0' | cut -d. -f1)
STYLE=$(echo "$input" | jq -r '.output_style.name // empty')

dir=$(basename "$DIR")

color_for() {
  local val="$1"
  if [ "$val" -ge 80 ]; then
    printf '\033[0;31m'
  elif [ "$val" -ge 50 ]; then
    printf '\033[0;33m'
  else
    printf '\033[0;32m'
  fi
}

style_part=""
if [ -n "$STYLE" ]; then
  style_part=" $(printf '\033[38;5;208m%s\033[0m' "$STYLE")"
fi

parts="$(printf '\033[0;35m%s\033[0m' "$dir") $(printf '\033[0;36m%s\033[0m' "$MODEL")${style_part}"

if [ -n "$USED" ]; then
  used_int=$(printf '%.0f' "$USED")
  color=$(color_for "$used_int")
  parts="$parts $(printf "${color}Context: ${used_int}%%\033[0m")"
fi

if [ -n "$FIVE_H" ]; then
  five_int=$(printf '%.0f' "$FIVE_H")
  color=$(color_for "$five_int")
  [ -n "$FIVE_RESET" ] && parts="$parts $(printf "${color}Reset: ${FIVE_RESET}\033[0m")"
  parts="$parts $(printf "${color}5h: ${five_int}%%\033[0m")"
fi

if [ -n "$SEVEN_D" ]; then
  seven_int=$(printf '%.0f' "$SEVEN_D")
  color=$(color_for "$seven_int")
  parts="$parts $(printf "${color}7d: ${seven_int}%%\033[0m")"
fi

printf "%s" "$parts"
```
:::

[starship](https://starship.rs/ja-JP/) や [oh my zsh](https://ohmyz.sh/) を参考に、Modsで作り直してみました

大きな違いは `button` を使ったことと、`hover` した際に表示を切り替えたことです

Modsで追加した機能は以下です

- ディレクトリ名をbuttonにして、クリックすると絶対パスをコピー
  - GitHub PR番号はクリックするとWebが開く
- 7dのリセット日時を表示
- nodeやpython等のバージョンを表示
  - starship [nodejs.rs](https://github.com/starship/starship/blob/main/src/modules/nodejs.rs) 等を参考に実装
- 他にも、手動リロードやGitのステータス、レビュー系のコマンドを実行するbuttonを表示

![3.png](/images/20261025_claude-code-mods/3.png)

@[card](https://github.com/Flupinochan/metalmental-claude/tree/master/plugins/statusline)

### 2. ui-plus

messageを開閉したい! tableをMarkdownでコピーしたい! という思いから作成しました

- userは青色、assistantは緑色で表示
- userとassistantのmessageを折りたためるbuttonに変更
  - 直近の2つのmessageのみ開き、古いmessageは自動で折りたたむ
  - 手動で開いたmessageは開いたまま
- tableやcode blockの上にMarkdown形式でコピー可能なbuttonを表示
- toolやsubagentの実行ログを非表示
  - 回答中は実行ログを表示しつつ、回答終了時に実行ログを非表示
  - Ctrl + o で標準のUIに切り替えて実行ログを表示可能

![4.png](/images/20261025_claude-code-mods/4.png)

@[card](https://github.com/Flupinochan/metalmental-claude/tree/master/plugins/ui-plus)

## おわりに

それぞれ1日で作れました!

TypeScriptの型定義に加え、丁寧な説明があり、何ができて何ができないのかをすぐに把握できたことが非常に助かりました

Sonnet 5.5のみでコーディングし、コードは1行も書いていません!

**Terminalは自然言語で自分の好みにカスタムできる時代になった** と感じました

自分にあったUIにカスタムして生産性を向上させましょう!

## 補足

- [Claude CodeのUI構成](https://code.claude.com/docs/ja/plugins/mods/interface)
- [UIで利用可能なElement](https://code.claude.com/docs/ja/plugins/mods/gallery#pick-an-element)

使ってはいないのですが、以下のModsは個人的にいいね! と感じました

@[card](https://github.com/NahumLitvin/prismantis)

@[card](https://github.com/oikon48/prompt-rail)

どのようなことができるのか、まだよく分からない方は参考にしてみてください

なお、Modsは**Sandboxで制限されない**ため、利用する際は注意してください
