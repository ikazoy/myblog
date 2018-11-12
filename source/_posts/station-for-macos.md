---
title: 指定のアプリをどこからでも開けるようにするとStationがもっと便利になる for MacOS
date: 2018-11-12T07:54:02.788Z
categories:
  - tool
author: ikazoy
layout: blog
---
# Stationとは

[Station](https://getstation.com/)

仕事で使用するアプリ（Gmail、カレンダー、Slack、Facebook messenger etc..)を統合して扱うためのアプリです。

日本語だとこちらの[ブログ](https://ksk1030.hatenablog.com/entry/messanger_apps)がまとまっててよさそうです。

Googleアカウント統合アプリ [Kiwi for Gmail](https://www.kiwiforgmail.com/)を現在進行系で使っていたのですが、同僚からのススメで乗り換えてみました。

ちなみに、過去[Franz](https://meetfranz.com/), [Shift](https://tryshift.com/)など類似のアプリを使っていたことがあります。

# Station内のアプリを最速で開きたい in MacOS

Stationにたくさんのアプリが統合できるとはいえ、それでもWebブラウザやターミナルなど別アプリからStationを開いて、目的のアプリに移るのは手間がかかります。

## 通常のステップ

1. Webブラウザを開いている状態
2. `Cmd + Tab`やSpotlightなどでStationを起動する
3. `Cmd + t`でStation内のQuick Switchを開く
4. 目的のアプリの名前を入れる（例：`Slack`)

## こうしたい
1. Webブラウザを開いている状態
2. ショートカットキー（ホットキー）を入力するとQuick Switchを開いた状態でStationが起動する
3. 目的のアプリの名前を入れる（例：`Slack`)

![station](/source/images/スクリーンショット 2018-11-12 17.15.10.png)

1ステップ減りました！1日数十回使いそうなので重要。

# どうやるか

AppleScriptをアプリケーション化して、ホットキーを割り当てます。

## AppleScript作成

1. スクリプトエディタを開く(アプリケーション→ユーティリティ）
2. 下記のコードを貼り付ける

    ```
    -- https://stackoverflow.com/questions/4242437/applescript-how-do-i-launch-an-application-and-then-execute-a-menu-command
    tell application "Station"
        activate
        tell application "System Events"
            tell process "Station"
                tell menu bar 1
                    tell menu bar item "View"
                        tell menu "View"
                            click menu item "Show Quick-Switch"
                        end tell
                    end tell
                end tell
            end tell
        end tell
    end tell
    ```
3. ファイル→書き出す→ファイルフォーマット：アプリケーションを選択→保存
    - アプリケーションの名前は`quick-switch-Station`などご自由に設定ください。

※スクリプトエディタからAppleScriptを実行するには、OSのシステム環境設定→セキュリティとプライバシー→アクセシシビリティから許可を与える必要があります。

※同様に、上記で保存したアプリケーションを起動する場合は、そのアプリケーションにアクセシビリティの許可を与える必要があります。

※AppleScriptは書いたことないので、ベターな書き方＆実行方法があれば教えてください。

## ホットキー割り当て

お好きな方法で`quick-switch-Station`を起動できるように設定すればよいです。

私は[CLCL](https://itunes.apple.com/jp/app/clcl-lite/id495511246?mt=12)を使って、optionキー2回押しで、`quick-switch-Station`が起動するように設定しております。

![clcl設定](/source/images/スクリーンショット 2018-11-12 17.49.22.png)

ユーザーではないので試してないですが、Alfredなどを使っても便利でしょう。
