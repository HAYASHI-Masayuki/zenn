---
title: "サプライチェーン攻撃対策に、safe-chainとSocket Firewall Freeを試してみた"
emoji: "⛓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["サプライチェーン攻撃", "security", "zennfes2025infra"]
published: true
---

前置き
================================================================================

2025年9月14日[^1]、npmで公開されているパッケージに対する攻撃があったようです。今までの同様の攻撃と比べて規模が大きく、よく利用されるパッケージも攻撃を受け、またマルウェアがワームとして広がったことから注目されている気がします。

パッケージ利用者の観点からは、「信頼しているパッケージが攻撃されて、新しいバージョンにマルウェアが混入した」という、一般的なサプライチェーン攻撃の流れです。

とはいえ最近はサプライチェーン攻撃の頻度も増加し、規模も大きくなっているように思え、現実的な脅威を感じます。

残念ながらサプライチェーン攻撃に対しては、Webアプリケーション開発におけるXSSに対する適切なエスケープ、SQLインジェクションに対するプレースホルダの使用、のような決定版的な対策は今のところなさそうです。

そんな中、「パッケージマネージャ用アンチウィルス」的なマルウェアブロックツールは、今後基本的な対策になっていくのではないかと考えています。

このようなツールとして、[safe-chain](https://github.com/AikidoSec/safe-chain)や[Socket Firewall Free](https://socket.dev/blog/introducing-socket-firewall)などがあるようです。

この記事では、この2ツールを簡単に試してみた結果について書きます。


[^1]: "On September 14, 2025, we were notified of the Shai-Hulud attack" https://github.blog/security/supply-chain-security/our-plan-for-a-more-secure-npm-supply-chain/


safe-chain
================================================================================

safe-chainは[Aikido Security](https://www.aikido.dev/)というベルギーのセキュリティ関係(多分)の会社が開発しているCLIツールです。v1.1.4現在、npmリポジトリのマルウェアのブロックに対応しています。


### 使い方

インストール・セットアップすると、シェルと統合し、 `npm` コマンド等をラップすることで動作します。

```bash
npm install -g @aikidosec/safe-chain
safe-chain setup
# ここでシェルを再起動する
npm install # 実際には `aikido-npm` が動作する
# あるいは `safe-chain setup` せず、
aikido-npm install # でもOK
```

GitHub Actions等のCI環境でも、ローカル開発環境同様、単にインストール・セットアップした上でラップされている `npm` 等を実行する形で動かせます。

```yaml
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # 以下のステップはREADME(https://github.com/AikidoSec/safe-chain/blob/5eedbfb57fb42998487f8d153c3600d2c6a443e1/README.md)からのコピペ
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"

      - name: Setup safe-chain
        run: |
          npm i -g @aikidosec/safe-chain
          safe-chain setup-ci

      # マルウェアが検出された場合、ここで落ちる
      - name: Install dependencies
        run: |
          npm ci
```


### 実際にマルウェアをブロックできるか

自由なソフトウェアなので、ソースコードで動作を確認することができます。マルウェアの判定は、READMEにも書かれているようにAikido Security自身の管理しているデータベースを使って行っているようです。ソースコードには実際に参照しているJSONファイルのURLも記載されているので、必要なときには確認することもできます。

検出テスト用にsafe-chainがマルウェア扱いするsafe-chain-testという無害なパッケージが用意されています。これをインストールしてみます。

```bash
$ npm install safe-chain-test
✖ Safe-chain: Malicious changes detected:
 - safe-chain-test@0.0.1-security

Exiting without installing malicious packages.
```

しっかりエラーになり、 `node_modules` ディレクトリ内を見てもインストールされていません。ブロックできたようです。


### 注意点

safe-chainはv1.1.0より以前は、ロックファイルを解析することでマルウェアのチェックを行っていたようです。v1.1.0以降はプロキシでネットワークを中継することでチェックするように変更され、同時にそれまで完全にはサポートされていなかった `npm` 以外の各コマンド、 `yarn` や `pnpm`, また `bun` に対応しました。ツールごとにフォーマットの違うロックファイルに対応するのは現実的ではなかったんでしょうね……。

各ツールに簡単に対応できる代わりに、実際にダウンロードが発生しない場合のブロックはできなくなっているので、そこには注意が必要かと思います。サプライチェーン攻撃を考慮すると、CI等でのパッケージマネージャのキャッシュ戦略も見直した方がよさそうです。


Socket Firewall Free
================================================================================

Socket Firewall Freeは、[Socket](https://socket.dev/)というこれまたセキュリティ関係(多分)の会社が開発しているCLIツールです。Enterprise版も今後提供予定[^2]とのことです。

自由なソフトウェアではなく、[PolyForm Shield License 1.0.0](https://polyformproject.org/licenses/shield/1.0.0/)[^3]という聞き慣れないライセンスで提供されています。このライセンスは対象のソフトウェアとなんらかの形で競合する用途では使用できないという特殊な制限を持つライセンスのようです。

今回、この記事における使用においては、それ自体がSocket Firewall Freeが提供するものと競合しないとう判断で使用しています。

safe-chainとの非常に大きな違いとして、npmリポジトリ以外にも対応しています。Free版でも、Python, Rustのパッケージリポジトリに対応しており、Enterprise版に関しては、 "All languages" と書かれています。[こちらのドキュメント](https://docs.socket.dev/docs/socket-firewall-overview)にあるように、

> JavaScript/TypeScript, Python, Go, Java (Maven, Gradle), Ruby (gem, Bundler), Rust (cargo), and .NET (NuGet)

に対応していると思われます。


### 使い方

safe-chainと違い、シェルとの統合はありません。 `sfw` コマンドの引数として実行したい `npm` 等のコマンドを渡すことで、マルウェアのブロックが動作する状態で `npm` 等のコマンドが実行されるようになります。

```bash
npm install -g sfw
sfw npm install
```

CI環境では、GitHub Actionsのアクションが開発元によって用意されているので、それを使うことができます(未テスト。サービスがどのような形態で提供されているかもわからないので)。

[SocketDev/action: GitHub Action to run Socket in CLI or Firewall mode](https://github.com/SocketDev/action)


### 実際にマルウェアをブロックできるか

すばらしいことに、safe-chainの検出テスト用のsafe-chain-testを、Socket Firewall Freeもブロックしてくれます。これは動作確認に非常に便利で助かります。

```bash
$ sfw npm install safe-chain-test
Protected by Socket Firewall
npm ERR! code E403
npm ERR! 403 403 Forbidden - GET https://registry.npmjs.org/safe-chain-test/-/safe-chain-test-0.0.1-security.tgz
npm ERR! 403 In most cases, you or one of your dependencies are requesting
npm ERR! 403 a package version that is forbidden by your security policy, or
npm ERR! 403 on a server you do not have access to.

npm ERR! A complete log of this run can be found in: /home/<USER>/.npm/_logs/2025-10-12T07_53_10_234Z-debug-0.log

=== Socket Firewall ===

failed to parse URL: registry.npmjs.org/http://registry.npmjs.org:443/-/npm/v1/security/advisories/bulk
failed to parse URL: registry.npmjs.org/http://registry.npmjs.org:443/-/npm/v1/security/audits/quick
 - blocked npm package: name: safe-chain-test; version: 0.0.1-security

For more information, check https://socket.dev.
```

ちょっと余計なログが残っていますが、正しく動作し、safe-chain-testはインストールされませんでした。


### 注意点

Socket Firewall Freeは機能的にはsafe-chain＋αという感じで、非常によさそうなのですが、とにかくライセンスに不安があります。最初からEnterprise版を検討する方がいいかもしれません。Enterprise版の契約でも、競合に関する条項が不要とは限らないので、やはり不安が残りますが。

また、Socket Firewall Freeは匿名の利用状況を収集しています。ただこれは許容できるのであれば、しっかり明記されていることはむしろ安心材料かもしれません。

もう1つ、safe-chainと違ってシェル統合がないため、単純に `npm` 等のコマンドの使用時に `sfw` とつけ忘れるともちろんマルウェア検知は働きません。とはいえsafe-chainも、シェルを経由しない環境ではやはりスルーしてしまいますし、こういう形態でのブロックにおける共通の課題と言えそうです。


[^2]: "In the near future, we will also be offering Socket Firewall Enterprise to our customers." https://socket.dev/blog/introducing-socket-firewall#Scaling-Protection-Across-Ecosystems
[^3]: 日本語での情報は非常に少ない。こちらの記事(https://tech.bitbank.cc/20210823/)は参考になるが、 "すべて、非営利での使用は許可しています。" としているのはおそらく間違えだと思うので、参考程度に。もちろん、私の記事についても参考程度に。


2ツールのざっくりとした比較
================================================================================

|                                            | safe-chain                                            | Socket Firewall Free                                    |
| ---                                        | ---                                                   | ---                                                     |
| 既存のパッケージマネージャをそのまま使える | ○                                                    | ○                                                      |
| シェル統合                                 | ○                                                    | ×                                                      |
| 対応言語(パッケージマネージャ)             | JavaScript(npm, yarn, pnpm, bun)                      | JavaScript(npm, yarn, pnpm)/Python(pip, uv)/Rust(cargo) |
| マルウェアデータベース                     | 自社管理のもの(https://intel.aikido.dev/?tab=malware) | 不明(多分こちらも自社で持ってるデータベースかと)        |
| ライセンス                                 | AGPL                                                  | PolyForm Shield License 1.0.0                           |

なんとなく比較してみましたが、基本的なところはかなり大きく違うので、あまり比較の意味がないですね……。まあsafe-chainがOSS寄りで、Socket Firewall Freeがプロプライエタリ寄り、というのがいちばん重要なところかと思います。


まとめ
================================================================================

どちらのツールも、適切に使用し、またマルウェアデータベースが十分に更新されていれば、サプライチェーン攻撃のリスク減少のために有用そうです。Socket Firewall Freeの方が高機能そうですが、ライセンス的に導入が難しい場合が多いかもしれません。Aikido Securityも、実はnpmリポジトリ以外のマルウェアのデータを持ってそうなので、safe-chainの対応言語の追加を期待するのもありかもしれません。


この記事のライセンス
================================================================================
![クリエイティブ・コモンズ表示4.0国際ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
*この文書は[CC BY(クリエイティブ・コモンズ表示4.0国際ライセンス)](http://creativecommons.org/licenses/by/4.0/)で公開します。*


