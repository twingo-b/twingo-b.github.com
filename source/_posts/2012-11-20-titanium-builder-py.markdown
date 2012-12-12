---
layout: post
title: "iOS builder.pyのコマンドラインオプション"
date: 2012-11-20 09:55
comments: true
categories: "Titanium iOS Android"
---
[Titanium mobile "early" Advent Calendar 2012](http://atnd.org/events/33731)用小ネタ、2個目です。

めずらしくiOSネタです＾＾；

開発環境はMacで、shellはzshを利用しています。

普段はMakeTiを使ってるんですが、Distributeとかもコマンドラインでやりたいなと思いました。そのためだけにTitanium Studio立ち上げるのめんどくさいですし―。

そんな時は直接builer.pyを動かす必要があるんですが、コマンドラインオプションを調べようとヘルプを見ると、こんな感じでオプションがよくわからない。。Androidのbuilder.pyはまだわかるのに。

```
~/Library/Application\ Support/Titanium/mobilesdk/osx/2.1.3.GA/iphone/builder.py
builder.py <command> <version> <project_dir> <appid> <name> [options]

available commands:

  install       install the app to itunes for testing on iphone
  simulator     build and run on the iphone simulator
  adhoc         build for adhoc distribution
  distribute    build final distribution bundle
  xcode         build from within xcode
  run           build and run app from project folder
```

どうしたもんかなーと思っていると、build.logの最初にそれっぽい情報が！！！

```
# AdHocビルドのbuild.logの例
cd ~/path/to/[your titanium mobile Project]
head -n 30 build/iphone/build/build.log
================================================================================
Appcelerator Titanium Diagnostics Build Log
The contents of this file are useful to send to Appcelerator Support if
reporting an issue to help us understand your environment, build settings
and aid in debugging. Please attach this log to any issue that you report.
================================================================================

Starting build at 11/20/12 12:00

Build details:

   timestamp=10/02/12 16:16
   module_apiversion=2
   version=2.1.3
   githash=15997d0


Script arguments:
   /Users/$USER/Library/Application Support/Titanium/mobilesdk/osx/2.1.3.GA/iphone/builder.py
   adhoc
   6.0
   ~/path/to/[your titanium mobile Project]
   [appid]
   [name]
   [xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx]
   [証明書の名前]
   universal
   /Users/$USER/Library/Keychains/login.keychain
```

では、オプションに順番に入れたら動くかなーと思ってやってみたら、動きました！

```
security unlock-keychain /Users/$USER/Library/Keychains/login.keychain
/Users/$USER/Library/Application\ Support/Titanium/mobilesdk/osx/2.1.3.GA/iphone/builder.py "adhoc" "6.0" "~/path/to/[your titanium mobile Project]" "[appid]" "[name]" "[xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx]" "[証明書の名前]" "universal" "/Users/$USER/Library/Keychains/login.keychain"
```

security unlock-keychain ~は、事前にやっとかないと、「SystemExit: 65」になりました。build.logの最後の方に「User interaction is not allowed.」と出てましたので判断できました。

ipaファイルは以下に出力されてました

```
~/path/to/[your titanium mobile Project]/build/iphone/build/Debug-iphoneos/[name].ipa
```

同じように、Distributeも試してみましたが、大丈夫そう！

これで、コマンドライン実行＆自動化できそうですー。

