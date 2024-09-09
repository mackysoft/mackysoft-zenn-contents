---
title: "【対策】CPU起因でVisualStudioがクラッシュおよびエラーを吐く可能性への対策について"
emoji: ":desktop_computer:"
type: "tech"
topics: ["visualstudio"]
published: false
#published_at: 2024-09-12 12:00
---
## はじめに

しばらく、VisualStudioのエラーに悩まされていた。

起動時にIntelliSenseが機能しなくなる、カラーテーマが機能しなくなる、使用中にフリーズ、クラッシュなど…。作業に集中できない問題が続いていたところ、ようやく「これが原因ではないか？」という点を特定できたので、記事に残します。

## 何が起きていたか

筆者が遭遇した主な現象は、起動時にVisualStudioがエラーを吐き、エディターの機能がほとんど機能しなくなるというものである。
エラーは以下のように、「内部エラーのため、機能'○○'は使用できません。」という文言とともに発生する。

![](https://storage.googleapis.com/zenn-user-upload/fa969221af0c-20240909.png)

スタックトレースを表示すると、以下のような例外が見ることができる。

> StreamJsonPrc.ConnectionLostException:リモートパーティとのJSON-RPC接続は、要求が完了する前に失われました。

![](https://storage.googleapis.com/zenn-user-upload/ef0906d884c5-20240909.png)

これらのエラーを基に行ったこれまでの試行錯誤をまとめると、

- VisualStudioの再インストールを試みても変わらない
- VisualStudioのバージョンは関係ない
- 規模の大きい開発プロジェクトでのみ起発生する。作ったばかりの小さいプロジェクトでは発生しない
- エディターの拡張機能は関係ない

以上のことまでは分かったのだが、核心的な原因は特定できなかった。

## ハードウェア要因？

同じような問題が起きていないかTwitterを眺めていたところ、ある記事がヒットした。

https://gigazine.net/news/20240227-intel-processor-game-app-crash/

どうやら、Intel製の一部のCPU起因によるアプリのクラッシュが報告されており、VisualStudioでも同様のクラッシュが確認されているという話である。

> 問題は「Core i9 13900K」や「Core i9 14900K」を搭載したマシンで多く確認されているほか、その他のIntel製CPUを搭載したマシンでも同様の問題が発生する可能性があるとされています。

> クラッシュ問題はOodleやゲームの実装に起因するものではなく、Intel製CPUのハードウェア的な問題が原因で発生しています。具体的には「問題が確認されているモデルの極一部の個体」が特定のクロックレートと消費電力に達した際にシステムが不安定になって誤った命令を実行することが原因とのこと。

> RAD Game Toolsによると、Intel製CPUの問題はOodleだけに影響するものではなく、「RealBench」「CineBench」「Prime95」「Handbrake」「Visual Studio」といったソフトウェアでも同様の問題に起因するクラッシュが確認されているとのこと。

ちなみに、筆者が使用しているCPUは「Core i9-14900KF」であり、報告があるシリーズと一致する。

これまでの状況を鑑みて「遭遇しているエラーの原因は、このCPUの問題なのではないか？」と考え、RADが推奨する対策を行うことにした。

## 対策をしてみる

RADは、いくつかの対策を推奨しているが、今回は「Intel Extreme Tuning Utility（XTU）」を使用し、パフォーマンスコアに制限をかけて対策を行った。

https://www.radgametools.com/oodleintel.htm

念のために断っておくと、この対策を行う場合に筆者は責任を負わない。

> その他の回避策として、チューニングユーティリティを使用するか、BIOS設定を変更する必要があります。誤って行うと、システムが損傷する可能性があることに注意してください。 ここで推奨する変更は、私たちの知る限りでは完全に安全ですが、損害や損失についてはお客様が単独で責任を負います これらの設定を工場出荷時のデフォルトから変更したことが原因です。

### 1. XTUをインストール

Intelが提供しているXTUをインストールする。

https://www.intel.com/content/www/us/en/download/17881/intel-extreme-tuning-utility-intel-xtu.html

### 2. BIOSで低電圧保護を有効にする

XTUを実行した後はポップアップが表示され、以下が求められるはずである。

> Ensure you have the latest BIOS patch, latest Windows update, and have Undervolt Protection enabled in BIOS

最新アップデートは行われているものとして、あとはBIOSでUndervolt Protection（低電圧保護）を有効にする必要がある。

コンピュータのBIOSを起動し、検索ボックスなどから「Undervolt Protection」の項目を探してEnabledとする。

![](https://storage.googleapis.com/zenn-user-upload/ba8b69263f4f-20240909.jpg)

これでBIOSの設定は完了。

### 3. XTUでパフォーマンスコア乗数をx55~x53まで下げる

XTUでPerformance Core Ratioを下げる。RADの報告ではx55~x53辺りまで下げると良いらしい。

> 多くの人にとって成功したと報告されている回避策は、Intel XTUを使用し、パフォーマンスコア乗数をx55からx54またはx53に下げることです。

筆者は、乗数をx54まで下げて適用した。

![](https://storage.googleapis.com/zenn-user-upload/ffcdf2af90c5-20240909.png)

以上で、VisualStudioのクラッシュ対策は完了である。

## 現時点で

XTUでの対応から１週間経った現時点では、今までのようなエラーは発生しておらず、VisualStudioで快適に開発を続けられている。
特に、今まではVisualStudioの起動時とコンパイル時にかなり激しくファンが回っていたのだが、対応以降は驚くほど静かである。

そもそもの問題の発生でのランダム性が強く、再現性に乏しいものであったことから、この対策によって完全に改善されたものかは断定できない。
ただ「コードベースが大規模なプロジェクトでのみエラーが発生する」という点とも辻褄が合うので、この対応は正しかったと考えている。

## 参考

- https://gigazine.net/news/20240227-intel-processor-game-app-crash/
- https://www.radgametools.com/oodleintel.htm