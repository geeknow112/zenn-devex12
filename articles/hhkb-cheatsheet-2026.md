---
title: "【HHKB】開発者のためのチートシート完全版｜Vim連携・Karabiner設定・4層カスタマイズまで徹底解説"
emoji: "⌨️"
type: "tech"
topics: ["hhkb", "vim", "karabiner", "keyboard", "開発環境"]
published: true
---

## はじめに

Happy Hacking Keyboard（HHKB）は、1996年の誕生以来、プログラマーに愛され続けている60%キーボードです。

「**馬の鞍は使い込むほど馴染む。キーボードも同じだ**」という開発者・和田英一先生の哲学が込められたこのキーボードは、一度慣れると他のキーボードには戻れなくなる中毒性があります。

この記事では、HHKBを**開発者目線**で徹底的に使いこなすための情報をまとめました。単なるキー配列表ではなく、Vim連携、Karabiner-Elements設定、HHKB Keymap Toolによる4層カスタマイズまで踏み込んで解説します。

## 目次

1. [HHKBレイアウトの設計思想](#hhkbレイアウトの設計思想)
2. [Fnキー組み合わせ完全リファレンス](#fnキー組み合わせ完全リファレンス)
3. [Vim/Neovimユーザーのためのカスタマイズ](#vimneovimユーザーのためのカスタマイズ)
4. [Mac開発者向けKarabiner-Elements設定](#mac開発者向けkarabiner-elements設定)
5. [Windows開発者向けAutoHotkey設定](#windows開発者向けautohotkey設定)
6. [HHKB Keymap Toolで4層カスタマイズ](#hhkb-keymap-toolで4層カスタマイズ)
7. [HHKB Studio ジェスチャーパッド活用術](#hhkb-studio-ジェスチャーパッド活用術)
8. [DIPスイッチ設定の最適解](#dipスイッチ設定の最適解)
9. [Bluetooth運用のベストプラクティス](#bluetooth運用のベストプラクティス)
10. [開発ワークフロー別おすすめ設定](#開発ワークフロー別おすすめ設定)

---

## HHKBレイアウトの設計思想

HHKBのレイアウトには明確な設計思想があります。

### Sun Type 3キーボードの継承

HHKBの配列はSun Microsystemsのワークステーション用キーボード「Sun Type 3」に強く影響を受けています。特に重要なのが以下の2点：

1. **Ctrlキーの位置**: CapsLockの位置にCtrlを配置
2. **Escキーの位置**: 数字の1の左側（`~`キーの位置）

これはUNIX/Linux環境での作業を想定した配置で、vim/Emacsユーザーにとって理想的です。

### キー配列の比較

```
一般的なキーボード:
[Esc] [F1] [F2] ... [F12]
[`~] [1] [2] [3] [4] [5] [6] [7] [8] [9] [0] [-] [=] [Backspace]
[Tab] [Q] [W] [E] [R] [T] [Y] [U] [I] [O] [P] [{] [}] [\]
[CapsLock] [A] [S] [D] [F] [G] [H] [J] [K] [L] [;] ['] [Enter]

HHKB:
[Esc/`~] [1] [2] [3] [4] [5] [6] [7] [8] [9] [0] [-] [=] [\] [`~]
[Tab] [Q] [W] [E] [R] [T] [Y] [U] [I] [O] [P] [{] [}] [Delete]
[Ctrl] [A] [S] [D] [F] [G] [H] [J] [K] [L] [;] ['] [Enter]
```

### なぜCtrlがAの隣なのか

ターミナル操作、vim、Emacsでは`Ctrl`キーを多用します。

| 操作 | 一般キーボード | HHKB |
|------|---------------|------|
| `Ctrl+C` | 小指を左下に移動 | 小指をAの左に置くだけ |
| `Ctrl+[` (vim Esc代替) | 両手が必要 | 左手だけで可能 |
| `Ctrl+P/N` (履歴移動) | 手首をひねる | ホームポジション維持 |

**ホームポジションから指を離さない**という設計思想が、長時間のコーディングでも疲れにくい理由です。

---

## Fnキー組み合わせ完全リファレンス

### ファンクションキー（F1〜F12）

| 組み合わせ | 出力 | 主な用途 |
|-----------|------|----------|
| `Fn` + `1` | F1 | ヘルプ |
| `Fn` + `2` | F2 | リネーム、IDE: Quick Fix |
| `Fn` + `3` | F3 | 次を検索 |
| `Fn` + `4` | F4 | アドレスバー、Alt+F4で終了 |
| `Fn` + `5` | F5 | 更新、IDE: デバッグ実行 |
| `Fn` + `6` | F6 | フォーカス移動 |
| `Fn` + `7` | F7 | IDE: ビルド |
| `Fn` + `8` | F8 | IDE: 次のエラーへ |
| `Fn` + `9` | F9 | IDE: ブレークポイント |
| `Fn` + `0` | F10 | IDE: ステップオーバー |
| `Fn` + `-` | F11 | 全画面、IDE: ステップイン |
| `Fn` + `=` | F12 | DevTools、IDE: 定義へ移動 |

### カーソル移動（最重要）

HHKBの真骨頂。ホームポジションから一切手を離さずにカーソル操作が可能です。

| 組み合わせ | 出力 | Vimとの対応 |
|-----------|------|------------|
| `Fn` + `;` | ← | `h` |
| `Fn` + `/` | ↓ | `j` |
| `Fn` + `[` | ↑ | `k` |
| `Fn` + `'` | → | `l` |
| `Fn` + `K` | Home | `0` / `^` |
| `Fn` + `,` | End | `$` |
| `Fn` + `L` | Page Up | `Ctrl+B` |
| `Fn` + `.` | Page Down | `Ctrl+F` |

:::message
**覚え方**: 右手のホームポジション周辺に配置されています。`;` `/` `[` `'`は右手小指〜人差し指の自然な動きで押せます。
:::

### 編集キー

| 組み合わせ | 出力 | 用途 |
|-----------|------|------|
| `Fn` + `BS` | Delete | 前方削除（カーソル右の文字を削除） |
| `Fn` + `Tab` | Caps Lock | 滅多に使わないが必要な時に |
| `Fn` + `I` | Insert | オーバーライトモード（滅多に使わない） |
| `Fn` + `` ` `` | Print Screen | スクリーンショット |
| `Fn` + `\` | Pause | 稀にデバッグで使用 |

### メディア・音量（HYBRID/Studio）

| 組み合わせ | 出力 |
|-----------|------|
| `Fn` + `A` | ミュート |
| `Fn` + `S` | 音量ダウン |
| `Fn` + `D` | 音量アップ |

---

## Vim/Neovimユーザーのためのカスタマイズ

### なぜHHKBとVimは相性が良いのか

1. **Escの位置**: `~`の位置にEscがあるため、ノーマルモードへの切り替えが楽
2. **Ctrlの位置**: `Ctrl+[`でEscの代替が可能、さらに楽
3. **hjkl移動**: Fnキー組み合わせがhjklに近い配置

### vimrcでの設定例

```vim
" HHKB環境での推奨設定

" Escの代替（Ctrl+[が楽になる）
inoremap jj <Esc>
inoremap jk <Esc>

" Ctrlがホームポジションにあるので、これらが使いやすい
nnoremap <C-j> <C-w>j
nnoremap <C-k> <C-w>k
nnoremap <C-h> <C-w>h
nnoremap <C-l> <C-w>l

" ターミナルモードでもCtrl+[でノーマルモードへ
tnoremap <C-[> <C-\><C-n>

" HHKBではF1-F12がFn必須なので、代替マッピング
" :helpを開く
nnoremap <leader>h :help<CR>
" Quick Fix（通常F2）
nnoremap <leader>r :lua vim.lsp.buf.rename()<CR>
```

### Neovimでの追加設定

```lua
-- init.lua

-- HHKBの矢印キー位置に合わせたウィンドウ移動
vim.keymap.set('n', '<C-;>', '<C-w>h', { desc = 'Move to left window' })
vim.keymap.set('n', "<C-'>", '<C-w>l', { desc = 'Move to right window' })
vim.keymap.set('n', '<C-[>', '<C-w>k', { desc = 'Move to upper window' })
vim.keymap.set('n', '<C-/>', '<C-w>j', { desc = 'Move to lower window' })

-- LSP関連（F12の代替）
vim.keymap.set('n', 'gd', vim.lsp.buf.definition, { desc = 'Go to definition' })
vim.keymap.set('n', 'gr', vim.lsp.buf.references, { desc = 'Show references' })
vim.keymap.set('n', 'K', vim.lsp.buf.hover, { desc = 'Hover documentation' })
```

---

## Mac開発者向けKarabiner-Elements設定

Karabiner-ElementsでHHKBをさらに強化できます。

### インストール

```bash
brew install --cask karabiner-elements
```

### 推奨Complex Modifications

`~/.config/karabiner/karabiner.json`に追加：

```json
{
  "title": "HHKB Enhancements",
  "rules": [
    {
      "description": "Caps Lock to Escape (単押し) / Control (長押し)",
      "manipulators": [
        {
          "type": "basic",
          "from": {
            "key_code": "caps_lock",
            "modifiers": { "optional": ["any"] }
          },
          "to": [
            { "key_code": "left_control" }
          ],
          "to_if_alone": [
            { "key_code": "escape" }
          ]
        }
      ]
    },
    {
      "description": "左◇キー単押しで英数、右◇キー単押しでかな",
      "manipulators": [
        {
          "type": "basic",
          "from": {
            "key_code": "left_command",
            "modifiers": { "optional": ["any"] }
          },
          "to": [
            { "key_code": "left_command", "lazy": true }
          ],
          "to_if_alone": [
            { "key_code": "japanese_eisuu" }
          ]
        },
        {
          "type": "basic",
          "from": {
            "key_code": "right_command",
            "modifiers": { "optional": ["any"] }
          },
          "to": [
            { "key_code": "right_command", "lazy": true }
          ],
          "to_if_alone": [
            { "key_code": "japanese_kana" }
          ]
        }
      ]
    },
    {
      "description": "Ctrl+H を Backspace に",
      "manipulators": [
        {
          "type": "basic",
          "from": {
            "key_code": "h",
            "modifiers": { "mandatory": ["control"] }
          },
          "to": [
            { "key_code": "delete_or_backspace" }
          ]
        }
      ]
    }
  ]
}
```

### おすすめの組み合わせ

| 設定 | 効果 |
|------|------|
| Caps Lock → Esc/Ctrl | 単押しでEsc、長押しでCtrl |
| 左◇ → 英数 | IME切り替えが楽に |
| 右◇ → かな | IME切り替えが楽に |
| Ctrl+H → Backspace | Emacs風削除 |
| Ctrl+M → Enter | Emacs風改行 |

---

## Windows開発者向けAutoHotkey設定

### AutoHotkey v2 設定例

```autohotkey
; HHKB_enhance.ahk
#Requires AutoHotkey v2.0

; Caps Lock を Ctrl/Esc として使う
*CapsLock::
{
    Send '{LCtrl down}'
}
*CapsLock up::
{
    if (A_PriorKey = 'CapsLock') {
        Send '{LCtrl up}{Esc}'
    } else {
        Send '{LCtrl up}'
    }
}

; 左Win単押しでIMEオフ、右Alt単押しでIMEオン
~LWin up::
{
    if (A_PriorKey = 'LWin') {
        Send '{vkF3sc029}'  ; IME OFF
    }
}
~RAlt up::
{
    if (A_PriorKey = 'RAlt') {
        Send '{vkF2sc070}'  ; IME ON
    }
}

; Ctrl+H で Backspace
^h::Send '{Backspace}'

; Ctrl+M で Enter
^m::Send '{Enter}'

; Alt+矢印でウィンドウのスナップ
!Left::Send '#Left'
!Right::Send '#Right'
!Up::Send '#Up'
!Down::Send '#Down'
```

---

## HHKB Keymap Toolで4層カスタマイズ

HHKB Professional HYBRID以降では、公式の「HHKB Keymap Tool」で高度なカスタマイズが可能です。

### 4層レイヤーの概念

```
デフォルト層（Layer 0）: 通常のキー入力
Fn1層（Layer 1）: Fnキーを押しながら
Fn2層（Layer 2）: Fn2キーを押しながら（要カスタマイズ）
Fn3層（Layer 3）: Fn3キーを押しながら（要カスタマイズ）
```

### 開発者向けおすすめレイヤー設定

**Layer 0（デフォルト）**: 標準のまま

**Layer 1（Fn）**: 標準のFn機能

**Layer 2（Fn2 = 右Altに割り当て推奨）**:
- `Q` → `!` (よく使う記号)
- `W` → `@`
- `E` → `#`
- `R` → `$`
- `T` → `%`
- `Y` → `^`
- `U` → `&`
- `I` → `*`
- `O` → `(`
- `P` → `)`

**Layer 3（Fn3 = 右◇に割り当て推奨）**:
- マクロや頻繁に使うショートカット
- 例: `G` → `Ctrl+Shift+G`（VSCodeのGit操作）

### Keymap Tool設定のエクスポート/インポート

設定はJSONファイルとして保存でき、複数環境で共有可能です。

```bash
# 設定ファイルの場所（Windows）
%USERPROFILE%\Documents\HHKB Keymap Tool\

# 設定ファイルの場所（Mac）
~/Library/Application Support/HHKB Keymap Tool/
```

---

## HHKB Studio ジェスチャーパッド活用術

HHKB Studioには両サイドにジェスチャーパッドが搭載されています。

### デフォルト設定

| 操作 | 機能 |
|------|------|
| 左パッド 上下スライド | スクロール |
| 左パッド 左右スライド | 音量調整 |
| 右パッド 上下スライド | スクロール |
| 右パッド 左右スライド | 画面明度 |

### 開発者向けカスタマイズ例

Keymap Toolで以下のように変更：

| 操作 | カスタム機能 |
|------|-------------|
| 左パッド 左右 | `Alt+Tab` / `Alt+Shift+Tab`（ウィンドウ切り替え） |
| 右パッド 左右 | `Ctrl+Tab` / `Ctrl+Shift+Tab`（タブ切り替え） |
| 左パッド 上下 | `Ctrl+PageUp/Down`（VSCodeのエディタグループ間移動） |
| 右パッド 上下 | 縦スクロール（デフォルト） |

### ポインティングスティックの活用

StudioにはThinkPad風のポインティングスティック（トラックポイント）も搭載。

- **DPI調整**: `Fn+Shift+U/I`で感度変更
- **ドラッグロック**: 中ボタンダブルクリックでドラッグ状態を維持
- **スクロール**: 中ボタン押しながらスティック操作

---

## DIPスイッチ設定の最適解

### Professional HYBRID / HYBRID Type-S

| SW | 推奨 | 理由 |
|----|------|------|
| SW1 | OFF (Win) または ON (Mac) | OS選択 |
| SW2 | OFF | Backspace/Deleteはデフォルトで |
| SW3 | **ON推奨** | DeleteキーをBackspaceにすると快適 |
| SW4 | OFF | 左◇はそのまま |
| SW5 | **ON推奨（Mac）** | ◇をCommandとして使う |
| SW6 | OFF | 省電力は有効のまま |

### 筆者の設定（Mac開発者）

```
SW1: ON  (Macモード)
SW2: OFF
SW3: ON  (DeleteをBackspaceに)
SW4: OFF
SW5: ON  (◇をCommandに)
SW6: OFF
```

---

## Bluetooth運用のベストプラクティス

### マルチデバイス運用

HHKB HYBRIDは最大4台のデバイスとペアリング可能。

```
スロット1: メインPC（Windows）
スロット2: MacBook
スロット3: iPad
スロット4: 予備
```

### 切り替えショートカット

| 操作 | コマンド |
|------|----------|
| デバイス1に切替 | `Fn + Ctrl + 1` |
| デバイス2に切替 | `Fn + Ctrl + 2` |
| デバイス3に切替 | `Fn + Ctrl + 3` |
| デバイス4に切替 | `Fn + Ctrl + 4` |
| USB接続に切替 | `Fn + Ctrl + 0` |

### 接続トラブル時の対処

```
1. ペアリングモードに入る: Fn + Q
2. 特定デバイスの削除: Fn + Q → Fn + Ctrl + BS + (1-4)
3. 全削除: Fn + Z + BS
4. OS切替: Fn + Ctrl + W (Win) / Fn + Ctrl + M (Mac)
```

### バッテリー持ちを良くするコツ

- 使用しない時は電源オフ（`Fn + Ctrl + P`で電源メニュー）
- USB充電しながらBluetoothは非推奨（バッテリー劣化）
- 省電力モードは有効のまま（SW6 = OFF）

---

## 開発ワークフロー別おすすめ設定

### Webフロントエンド開発（VSCode）

```
重視ポイント:
- F12（定義へ移動）の多用 → Fn+= を覚える
- DevTools（F12）→ Ctrl+Shift+I も便利
- マルチカーソル（Ctrl+Alt+↓）→ Ctrl+Alt+Fn+/ で可能

Karabiner/AHK追加設定:
- Ctrl+; → 左のエディタへ
- Ctrl+' → 右のエディタへ
```

### バックエンド開発（ターミナル中心）

```
重視ポイント:
- Ctrl+C, Ctrl+D, Ctrl+Zを多用
- 履歴検索 Ctrl+R
- Ctrl+A（行頭）, Ctrl+E（行末）

すべてホームポジションCtrlで快適に操作可能。
```

### DevOps/インフラ（SSH多用）

```
重視ポイント:
- tmux操作（Ctrl+B プレフィックス）
- vim操作（Esc, :wq など）

HHKBならCtrl+B がホームポジション維持で押せる。
```

---

## まとめ：HHKBを最大限活用するために

### 1週間で慣れる練習ポイント

| 日 | 重点項目 |
|----|----------|
| 1-2日目 | Ctrlの位置に慣れる（Ctrl+C, Ctrl+Vなど） |
| 3-4日目 | 矢印キー（Fn+;/'[/]）を体に覚えさせる |
| 5-6日目 | Page Up/Down, Home/Endを使う |
| 7日目 | F1-F12をFn経由で使う |

### 設定の優先順位

1. **まず素の状態で1週間使う**（基本を体に覚えさせる）
2. **DIPスイッチを調整**（SW3, SW5あたり）
3. **Karabiner/AHKで微調整**（IME切替など）
4. **Keymap Toolで独自レイヤー**（慣れてから）

### 参考リンク

- [HHKB公式サイト](https://happyhackingkb.com/)
- [Keymap Toolダウンロード](https://happyhackingkb.com/download/)
- [Karabiner-Elements](https://karabiner-elements.pqrs.org/)
- [AutoHotkey](https://www.autohotkey.com/)

---

HHKBは「最初の1週間」を乗り越えると、手放せなくなるキーボードです。この記事がその助けになれば幸いです。
