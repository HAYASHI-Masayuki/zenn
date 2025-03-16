---
title: "逆引きWezTerm設定"
emoji: "💲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wezterm"]
published: true
---

前置き
================================================================================

最近、WindowsではTera Termから、MacではiTerm2から、それぞれ、WezTermに乗り換えました。Tera Term, iTerm2いずれも大きな困りごとはなかったのですが、Tera Termはtrue color未サポート、iTerm2では、ウィンドウのアクティブ化がちょっと重い気がする、などもあり、乗り換えてみまいた。

true colorで表示されるVim/Neovimのテーマはいい感じですし、なにより設定をテキストで書けるのがとてもよい体験で、気に入ってます。ウィンドウのアクティブ化が重い云々はあまり変わってない気がする。

とはいえ移行するための設定ではいろいろ苦労しました。せっかくなので苦労した個所を、逆引きリファレンスみたいな形で残しておきます。誰かのなにかの参考になれば。

*(この記事は1項目未実装です。気が向いたら書きます。)*


逆引きWezTerm設定
================================================================================

起動時のウィンドウ位置を指定する
--------------------------------------------------------------------------------

単純に設定で対応することはできないようです。`gui-startup`にフックして、`wezterm.mux.spawn_window`で位置を渡す必要があるようです。

```lua
wezterm.on('gui-startup', function()
    wezterm.mux.spawn_window({
        position = {
            x = 200,
            y = 100,
        },
    })
end)
```

参考: [Set initial window position before a window is created · Issue #2976 · wezterm/wezterm](https://github.com/wezterm/wezterm/issues/2976#issuecomment-1419492777)


起動時のウィンドウサイズを指定する
--------------------------------------------------------------------------------

文字セル数単位でサイズを指定する形でよければ、`initial_cols`, `initial_rows`を指定するだけです。

```lua
config.initial_cols = 160
config.initial_rows = 40
```

私の好み的にはこれで問題ないのですが、本当にウィンドウのサイズを明示的に指定したい場合の方法は、調べた限り見つかりませんでした。


### 画面一杯にする

画面一杯にする場合、Macでは`initial_cols`, `initial_rows`の値を十分に大きくすることでも可能ですが、Windowsではメモリを異常に消費し、起動に時間がかかるなど現実的ではなさそうでした。

そのようなことをしなくとも、普通に設定する方法がありました。[起動時のウィンドウ位置を指定する](#起動時のウィンドウ位置を指定する)同様、`gui-startup`にフックします。

```lua
wezterm.on('gui-startup', function()
    local _, _, window = wezterm.mux.spawn_window({})
    window:gui_window():maximize()
end)
```

参考: [maximize - Wez's Terminal Emulator](https://wezterm.org/config/lua/window/maximize.html)

ただしこの方法は、今度はMacで起動時にアニメーションが入ってしまいます。私はWindowsでは`maximize`を、Macでは`initial_cols`, `initial_rows`を使うようにしています。


### フルスクリーンにする

[画面一杯にする](#画面一杯にする)での`maximize`の代わりに`toggle_fullscreen`を使います。

```lua
wezterm.on('gui-startup', function()
    local _, _, window = wezterm.mux.spawn_window({})
    window:gui_window():toggle_fullscreen()
end)
```

参考: [toggle_fullscreen - Wez's Terminal Emulator](https://wezterm.org/config/lua/window/toggle_fullscreen.html)


TODO: ウィンドウの見た目をカスタマイズする
--------------------------------------------------------------------------------

`window_decorations`, `window_frame`, `window_padding`辺りについて、スクリーンショットつきで書きたい。


Windowsで、WSLにSSHで接続する
--------------------------------------------------------------------------------

WSLには、`config.default_domain = 'WSL:Ubuntu'`のような形で`wslhost.exe`を通して接続することもできますが、SSHで接続した方が速い気がします。私の環境ではシェルが初期化されて入力を受け付けるようになるまでの時間が1秒くらい違いました。

```lua
config.default_domain = 'wsl-ssh' -- `config.ssh_domains.name`に指定したものを
config.ssh_backend = 'Ssh2'
config.ssh_domains = {
    {
        name = 'wsl-ssh', -- なんでもいい
        remote_address = 'localhost:22', -- portを変えたりしている場合はそれに合わせる
        username = <USERNAME>,
        multiplexing = 'None',
    },
}
```

上記のように設定することで、起動時にWSLに接続します。この辺の設定にはいくつか罠があるのでざっくり説明します。


### `config.ssh_backend = 'Libssh'`(デフォルト値)はSSHエージェントに対応していない

`Libssh`の方が、暗号回りのサポートがよかったりするようで、そのためにデフォルトはこちらになっているようです。が、`Libssh`はSSHエージェントに対応していないため、SSHエージェントを使って接続したい場合は`Ssh2`を使うしかありません。

[Usage of `wezterm ssh` with ssh-agent on Windows 11 · wezterm/wezterm · Discussion #3772](https://github.com/wezterm/wezterm/discussions/3772#discussioncomment-7201688) こちらの話が少し関連していますが、私は特にWindows側の`~/.ssh/config`は設定していません。

なおこれだけではSSHエージェント転送は動かないはずです。エージェント転送もしたい場合は、[wsl2-ssh-agent](https://github.com/mame/wsl2-ssh-agent)を使うのが私の知る限り現時点ではベストそうです。(ただし、Pageantとは連携できなそう。)

参考:
[ssh_backend - Wez's Terminal Emulator](https://wezterm.org/config/lua/config/ssh_backend.html)
[wsl2-ssh-agent: WSL2からssh-agent.exeサービスへのブリッジ](https://zenn.dev/mametter/articles/49a2b505ec0275)


### `multiplexing = 'None'`を設定しないと接続時に落ちる

これを設定しないと、接続先にWezTermが必要なようです。

参考: [object: SshDomain - Wez's Terminal Emulator](https://wezterm.org/config/lua/SshDomain.html)


Macで日本語IME(AquaSKK/macSKK)を使う
--------------------------------------------------------------------------------

iTerm2では、AquaSKK用の設定があり、それをオンにするだけで一通り入力に問題はなかったのですが……。WezTermではそのままではAquaSKK/macSKKはうまく動作しないようです。変換中もCtrlとのコンビネーションキーがWezTerm側に吸われて、まともに変換できません。

以下のように設定することで対応できます。

```lua
config.macos_forward_to_ime_modifier_mask = 'SHIFT|CTRL'
```

参考: [macos_forward_to_ime_modifier_mask - Wez's Terminal Emulator](https://wezterm.org/config/lua/config/macos_forward_to_ime_modifier_mask.html)

ちなみにWezTermの日本語入力について調べるとよく、`config.use_ime`の話が出てきますが、この設定はだいぶ前にデフォルトでオンになったようで、今は気にする必要はなさそうです。


その他Macでのキーボード入力を調整する
--------------------------------------------------------------------------------

Macでは、IMEでのCtrl問題以外にもいろいろなキーがうまく働かないようです。私は以下
の3設定で回避しています。

```lua
config.keys = {
  -- Ctrl+Qが効かない
  { mods = 'CTRL', key = 'q', action = wezterm.action.SendString('\x11') },
  -- Ctrl+/はCtrl+_にしたい
  { mods = 'CTRL', key = '/', action = wezterm.action.SendString('\x1f') },
  -- \で¥が出るだけでなく、VimのLeaderで\を使いたいような場合も効かない
  { key = '¥', action = wezterm.action.SendString('\\') },
}
```

参考:
[Wezterm preventing `CTRL` + `/` from entering vim · Issue #3180 · wezterm/wezterm](https://github.com/wezterm/wezterm/issues/3180#issuecomment-1490214225)
[Unable to type backslash on mac · Issue #4051 · wezterm/wezterm](https://github.com/wezterm/wezterm/issues/4051)

正直この辺はだいぶややこしく、回避はしてますがこの設定が正しいかは怪しいです。


MacでiTerm2の`v|i`同様のフォント間スペーシングの調整をする
--------------------------------------------------------------------------------

私はWezTermでは[UDEV Gothic JPDOC](https://github.com/yuru7/udev-gothic)を使っています。このフォントをMacで表示すると、横が詰まって見えるので、iTerm2ではフォント設定の`v|i`みたいに書いてある個所(あれなんて意味でしょうね……？)を`100`から`101`にすることで調整していました。

これと同等のことをWezTermで行うには、`config.cell_width`を設定します。

```lua
config.cell_width = 1.1 -- これでiTerm2でのv|i = 101と同等
```

`config.font`に`stretch`を指定してフォントを設定することでも同様のことができるようですが、私の環境では動作しませんでした。フォントに依存するのかもしれません。

参考:
[cell_width - Wez's Terminal Emulator](https://wezterm.org/config/lua/config/cell_width.html)
[wezterm.font - Wez's Terminal Emulator](https://wezterm.org/config/lua/wezterm/font.html)


この記事のライセンス
================================================================================
![クリエイティブ・コモンズ表示4.0国際ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
*この文書は[CC BY(クリエイティブ・コモンズ表示4.0国際ライセンス)](http://creativecommons.org/licenses/by/4.0/)で公開します。*


