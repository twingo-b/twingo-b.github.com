---
layout: post
title: "Titanium MobileでDeployGateを利用する"
date: 2012-10-01 00:00
comments: true
categories: "Titanium Android DeployGate"
---
[@astronaughts](https://twitter.com/astronaughts)さんから依頼を受け、moduleを作成したところ、ひと通りうまく動作しました＾＾。

検証中にちょっとハマったところがあり、[@DeployGate_ja](https://twitter.com/DeployGate_ja)さんにドキュメントの改善要望をお問い合わせしました。

その後、こんなお話をいただいたので、Octpressの練習も兼ねてmoduleの作成方法を公開します。

{% img /images/tideploygate/deploygate_ja.png %}

サンプルソースはGitHubに置いておきましたので、[こちら](https://github.com/twingo-b/tideploygate)をどうぞ。

[Andorid SDK](http://developer.android.com/sdk/index.html)、[Titanium Studio](http://www.appcelerator.com/platform/titanium-studio/)、[DeployGate](https://deploygate.com/)で必要なものはインストール済みの前提です。

開発環境はMacで、shellはzshを利用しています。

## Titanium Mobileのandroid moduleの作成
### ~/.zshrc例
{% gist 3805894 %}

### moduleを作成する
適当なディレクトリに移動して、`.zshrc`でalias指定した`titanium`コマンドでmoduleのテンプレートを生成します。
```
$ titanium create --platform=android --type=module --name=TiDeployGate --id=local.twingob.android.tideploygate --android=$ANDROID_SDK
```
`Created android module project`と出れば成功です。

`tideploygate`ディレクトリ以下にテンプレートが作成されました。
```
LICENSE
assets
- README
bin
build
build.properties
build.xml
dist
documentation
- index.md
example
- app.js
hooks
- README
- add.py
- install.py
- remove.py
- uninstall.py
lib
libs
manifest
platform
- README
src
- local
  - twingob
      - android
          - tideploygate
              - ExampleProxy.java
              - TideploygateModule.java
timodule.xml
```

`build.xml`,`build.properties`がローカルPATHを指しており、そのままではコード共有しにくいので修正します。Titanium Studio、Eclipseなどを利用する場合は`.classpath`も同様に修正したほうが良いです。

### build.xmlの修正
build.propertiesから環境変数を参照できるように`<property environment="env"/>`を追加しました。
{% gist 3805994 %}

### build.propertiesの修正
{% gist 3806005 %}

### timodule.xmlの修正
example/app.jsからの動作確認時、リモートLogCatが利用できるように`<uses-permission android:name="android.permission.READ_LOGS" />`を記載しておきます。

`AndroidManifest.xml`はこの記述から自動生成されます。
{% gist 3806560 %}

### libディレクトリにdeploygatesdk.jarをコピーする
後ほど説明するantビルド時に、`lib`ディレクトリのjarが参照されます。

## moduleのソースコード変更、DeployGateの呼び出し
コマンドから自動生成されるソースコードは下記２種類です。
`src/local/twingob/android/tideploygate`
```
- ExampleProxy.java
- TideploygateModule.java
```

### 不用なサンプルファイルを削除
今回はProxyは利用しないので、`ExampleProxy.java`は削除します。

以降は、テンプレートをTitanium Studio、Eclipseにインポートしたほうがやりやすいかと思います。

### TideploygateModule.javaの名前が気になるのでリファクタリング
TiDeployGateModule.javaに修正しました。

### TiDeployGateModule#onAppCreate()にDeployGate#install()を追加
*** ここがTitanium MobileでDeployGateを利用する際のポイントになります。 ***

TiDeployGateModule#onAppCreate()はApplication#onCreate()時に呼び出されます。

また、Titanium Mobileで生成される`AndroidManifest.xml`の`android:debuggable`は常に`false`となっています。

そのため、`DeployGatei#install(Application app)`ではなく、`DeployGate#install(Application app, DeployGateCallback callback, boolean forceApplyOnReleaseBuild)` を利用する必要があります。`deploygatesdk-javadoc.jar`にその旨記載がありました。

例では、`forceApplyOnReleaseBuild:true`となるように、`DeployGate.install(app,null,true)`と指定しました。
{% gist 3806688 %}
Android SDKをEclipseから開発ビルドした場合は、`android:debuggable`は`true`になるみたいです。最初DeployGateのWeb上のドキュメントだけ見て、JavaDocを読んでなかったので、なぜ動かないのかわからず、ちょっとはまりました＾＾；。ドキュメントは最初にひと通り読まないとダメですよね＞＜。

### クラスのコンストラクタ以外のmethodサンプルを削除して、必要なmethodを追加
こちらはmoduleやDeployGateのサンプルの通りです。
{% gist 3806719 %}

### example/app.jsを記載して、moduleの動作確認を行う
`ant run`で動作確認できるように、js側でmoduleを呼び出します。
{% gist 3806739 %}

moduleは、テンプレートのディレクトリ直下でantコマンドを実行すると、emulatorで動作確認ができるので、実行してみます。

Android emulatorの起動
```
$ ant run.emulator
```
Terminalの別のTabでテストビルド
```
$ ant run
```
無事アプリは起動したのですが、emulatorのLogCatに下記のように出力され、動作しませんでした。DeployGateのクライアントアプリのインストールが必要ということですよね。
```
DeployGate  V  DeployGate is not available on this device.
```

そこで、`ant run`してできたapkを`dgate push`して、実機で動作確認して見ることにしました。`ant run`のログの最後の方に下記のように出力されているので、それを使ってみます。
```
     [exec] [DEBUG] /Users/[user]/android-sdk/platform-tools/adb -e install -r /var/folders/3j/cjcy2jcd6tlbkqyv1crfj1yh0000gn/T/m0sWWDXti/tideploygate/build/android/bin/app.apk  
     [exec] [INFO] Launching application ... tideploygate
```

```
$ dgate push /var/folders/3j/cjcy2jcd6tlbkqyv1crfj1yh0000gn/T/m0sWWDXti/tideploygate/build/android/bin/app.apk
Push app file successful!

Name :          tideploygate
Owner :         twingo_b
Package :       local.twingob.android.tideploygate
Revision :      1
URL :           https://deploygate.com/users/twingo_b/apps/local.twingob.android.tideploygate?key=3cc6266d2d8a606a25dfcb209331bf5de954734a
```

無事Pushできたので、アプリを実機で起動してみます。

動いた！

{% img /images/tideploygate/deploygatelog.png %}

リモートLogCatも動いた！

{% img /images/tideploygate/remotelogcat.png %}

うまく動いてますね＾＾。


## Titanium Mobileプロジェクトでの利用
### 実際のプロジェクトで利用するために、moduleのzipファイルを生成
`ant dist`コマンドで、zipファイルを生成します。
```
$ ant dist
...
zip.libs:
      [zip] Updating zip: /Users/[user]/[path to dir]/tideploygate/dist/local.twingob.android.tideploygate-android-0.1.zip

zip.metadata:

post.dist:

BUILD SUCCESSFUL

```
`dist`ディレクトリ以下にできたzipファイル`ocal.twingob.android.tideploygate-android-0.1.zip`をTitanium Mobileプロジェクト直下にコピーします。

### tiapp.xmlの修正
moduleの指定を追加します。リモートLogCatを利用する場合は、`<uses-permission android:name="android.permission.READ_LOGS" />`も記載します。
{% gist 3806841 %}

*** 実際には、Distributeビルド（リリースビルド）の際、これらの記載は削除し、module自体もビルドに含めないように、別途ビルドスクリプトを作成しています。 ***

## さいごに
DeployGateとTitanium Mobileの組み合わせは、開発をさらに加速できそうですよね！今後は[@astronaughts](https://twitter.com/astronaughts)さんに使い倒してもらいますw
