---
title: "Windows Terminal で現在のディレクトリを引き継いでペインやタブを複製する"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Windows", "WindowsTerminal"]
published: true
---

:::message
本記事で紹介する機能は [Windows Terminal 1.6](https://github.com/microsoft/terminal/releases/tag/v1.6.10571.0) 以降のバージョンで利用できます
:::

Windows Terminal で作業しているときに、ペインやタブを開いたあと、元いたのと同じディレクトリに移動（`cd`）するという作業をしたことはないでしょうか。たとえば、テストを実行している間にコードの変更をコミットしておこうというようなケースです。

この記事で紹介する設定を行うと、**元いたディレクトリを引き継いだペインやタブを開く**ことができるようになります。

ディレクトリのパスをコピペしたり、`cd` した履歴を漁ったりする作業から開放されます！

**設定前**

![](https://storage.googleapis.com/zenn-user-upload/k3ftr3g553ltjf2tb4kmnqfqxa6r)

**設定後**

![](https://storage.googleapis.com/zenn-user-upload/8o5j6gkahtygv03p6s433jzdk1ml)

下部に新しく開いたペインのカレントディレクトリが元のカレントディレクトリを引き継いで `~/workspace` になっています。

# 設定する

シェルの世界から**エスケープシーケンスを出力して Windows Terminal に対してカレントディレクトリを通知する**という設定が必要です。このエスケープシーケンスの出力はシェルごとに個別に設定する必要があります。

以降でシェルごとの設定方法を示します。

## bash - Cygwin/MSYS2

`.bashrc` に下記を追記します。

```bash
function _windows_terminal_osc_9_9 {
    # Inform Terminal about shell current working directory
    # see: https://github.com/microsoft/terminal/issues/8166
    printf '\e]9;9;%s\e\' "$(cygpath --windows "$(pwd)")"
}
PROMPT_COMMAND="_windows_terminal_osc_9_9; ${PROMPT_COMMAND}"
```

MSYS2 を使っていて `msys2_shell.cmd` を使ってシェルを起動するよう設定している場合は、`-here` オプションの指定が必要です。

**例：settings.json（Windows Terminal）**
```json
"commandline": "C:/msys64/msys2_shell.cmd -no-start -defterm -msys2 -here -shell bash",
```

## bash - WSL2

`.bashrc` に下記を追記します。

```bash
function _windows_terminal_osc_9_9 {
    # Inform Terminal about shell current working directory
    # see: https://github.com/microsoft/terminal/issues/8166
    printf '\e]9;9;%s\e\' "$(wslpath -w "$(pwd)")"
}
PROMPT_COMMAND="_windows_terminal_osc_9_9; ${PROMPT_COMMAND}"
```

## zsh - Cygwin/MSYS2

`.zshrc` に下記を追記します。

```bash
function _windows_terminal_osc_9_9 {
    # Inform Terminal about shell current working directory
    # see: https://github.com/microsoft/terminal/issues/8166
    printf '\e]9;9;%s\e\' "$(cygpath --windows "$(pwd)")"
}
precmd_functions+=(_windows_terminal_osc_9_9)
```

MSYS2 を使っていて `msys2_shell.cmd` を使ってシェルを起動するよう設定している場合は、`-here` オプションの指定が必要です。

**例：settings.json（Windows Terminal）**
```json
"commandline": "C:/msys64/msys2_shell.cmd -no-start -defterm -msys2 -here -shell zsh",
```

## zsh - WSL2

`.zshrc` に下記を追記します。

```bash
function _windows_terminal_osc_9_9 {
    # Inform Terminal about shell current working directory
    # see: https://github.com/microsoft/terminal/issues/8166
    printf '\e]9;9;%s\e\' "$(wslpath -w "$(pwd)")"
}
precmd_functions+=(_windows_terminal_osc_9_9)
```

## PowerShell

プロファイルを作成して次の関数を追記します。プロファイルの作成方法は次の記事などを参考にしてください。

[PowerShellでプロファイルを作成する方法 | クソざこCoding](https://www.zacoding.com/post/powershell-profile/)

```powershell
function prompt {
    $p = $($executionContext.SessionState.Path.CurrentLocation)
    $converted_path = Convert-Path $p
    $ansi_escape = [char]27
    "PS $p$('>' * ($nestedPromptLevel + 1)) ";
    Write-Host "$ansi_escape]9;9;$converted_path$ansi_escape\"
}
```
※ このコードは https://gist.github.com/skyline75489/480d036db8ae9069b7009377e6eebb79 から引用しました。

# 仕組み

リリースノートには、[ConEmu](https://conemu.github.io/) 独自のエスケープシーケンス `OSC 9;9;<windows path>`（OSC: Operating system commands）をサポートしたと記載されています。（ConEmu は Windows 専用のターミナルエミュレータのひとつです）

>  The terminal now supports [ConEmu’s OSC 9;9 sequence](https://conemu.github.io/en/AnsiEscapeCodes.html#ConEmu_specific_OSC), which sets the current working directory. If you emit `OSC 9;9;<Windows path>`, creating a duplicate of that pane or tab will use the Windows path you specified (Thanks [@skyline75489!](https://github.com/skyline75489)).
>
> [Windows Terminal Preview 1.6 Release | Windows Command Line](https://devblogs.microsoft.com/commandline/windows-terminal-preview-1-6-release/#:~:text=The%20terminal%20now,@skyline75489!)

エスケープシーケンスとはターミナルに表示する文字の色を変更したり、カーソルの位置を変更するといった、ターミナルの制御をするのに用いる特殊な文字列です。エスケープシーケンスそのものはターミナル上に文字として表示されません。

より詳しい説明：[ANSIエスケープコード - コンソール制御 - 碧色工房](https://www.mm2d.net/main/prog/c/console-02.html)

前章で紹介した設定ではターミナルにプロンプト（文字を入力するところ）を表示するタイミングで毎回 `OSC 9;9;<Windows path>` シーケンスを出力するような設定になっています。もし `cd` でディレクトリを移動したとしても、プロンプトを表示するタイミングで移動後のディレクトリがターミナルに伝わるため、ペインを分割したりする際に `cd` した後のディレクトリが引き継がれます。
