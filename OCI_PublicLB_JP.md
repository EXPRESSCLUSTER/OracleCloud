Oracle Cloud クラスタ (パブリックロードバランサ使用)
===

はじめに
---
本ガイドでは、Oracle Cloud の Computeインスタンス を使用した、CLUSTERPRO X for Linux/Windows のミラーディスク型クラスタを構築する手順について説明します。  
以下ではパブリックロードバランサを用いたクラスタ構成について説明します。

CLUSTERPRO X の詳細については、[こちら](https://jpn.nec.com/clusterpro/clpx/index.html)をご参照ください。


構成
---
本構成では、2node構成のミラーディスク型クラスタ(以下 Node1 / Node2) を構築し、Block Storageのデータはノード間で同期します。  
また、クラスタの現用系と待機系は、Oracle Cloud が提供するロードバランサーにおけるヘルス・チェックを利用して切り替えます。  
クライアントアプリケーションは、ロードバランサのIPアドレスを指定することで、仮想クラウド・ネットワーク(VCN)内のインスタンスにアクセスすることが可能となります。    

<div align="center">
<img src="https://user-images.githubusercontent.com/52775132/62447475-68a7f980-b7a0-11e9-9683-19133c4eabdd.png">
</div>


### 使用ソフトウェア
- Linuxの場合
  - Cent OS 6.10 (2.6.32-754.14.2.el6.x86_64)
    もしくは
    Cent OS 7.6 (3.10.0-957.12.2.el7.x86_64)
  - CLUSTERPRO X 4.1 for Linux (内部バージョン：4.1.1-1)
- Windowsの場合
  - Windows Server 2016 Standard
  - CLUSTERPRO X 4.1 for Windows (内部バージョン：12.11)


### クラスタ構成
- グループリソース
  - ミラーディスクリソース
  - Azure プローブポートリソース
- モニタリソース
   - Linuxの場合
     - ミラーディスクコネクトモニタリソース
     - ミラーディスクモニタリソース
     - Azure プローブポートモニタリソース
     - Azure ロードバランスモニタリソース
     - カスタムモニタリソース(NP解決用)
     - IPモニタリソース(NP解決用)
     - マルチターゲットモニタリソース(NP解決用)
   - Windowsの場合
     - ミラーコネクト監視リソース
     - ミラーディスク監視リソース
     - Azure プローブポート監視リソース
     - Azure ロードバランス監視リソース
     - カスタム監視リソース(NP解決用)
     - IP監視リソース(NP解決用)
     - マルチターゲット監視リソース(NP解決用)

Oracle Cloud の設定
---
1. インスタンスを作成する
   - 拡張オプションでフォルト・ドメインを分ける
     - Node1
        - 可用性ドメイン：可用性ドメイン1 (oIJw:AP-TOKYO-1-AD-1) 
        - フォルト・ドメイン：FAULT-DOMAIN-1
        - パブリックIPアドレス：10.0.0.8
        - プライベートIPアドレス：10.0.10.8
     - Node2
        - 可用性ドメイン：可用性ドメイン1 (oIJw:AP-TOKYO-1-AD-1)
        - フォルト・ドメイン：FAULT-DOMAIN-2
        - パブリックIPアドレス：10.0.0.9
        - プライベートIPアドレス：10.0.10.9
1. ブロック・ボリュームを作成する
   - 2ノード分のブロック・ボリュームを作成
1. インスタンスにブロック・ボリュームをアタッチする
   - Linuxの場合
     - デバイス・パス(/dev/oracleoci/oraclevdb)を選択する
   - iSCSIコマンドによりアタッチする
1. ロード・バランサの作成
   - 可視性タイプの選択で「パブリック」を選択
   - ネットワーキングの選択 - サブネットの指定でパブリックサブネットを選択
   - バックエンドの追加
     - クラスタ対象のサーバを(Node1, Node2)選択して追加
     - ポート：8080  (業務を提供しているポート番号：クラスタ側)
   - ヘルス・チェック・ポリシーの指定
     - プロトコル：TCP
     - ポート：26001
     - 間隔(ミリ秒)：5000
     - タイムアウト(ミリ秒)：3000
     - 再試行回数：2
   - リスナーの設定
     - トラフィックのタイプ：TCP
     - リスナーでモニターするポート：80  (業務を提供しているポート番号：クライアント側)
1. 必要に応じてセキュリティリストを設定

CLUSTERPRO のインストールと設定
---
※記載がない値は既定値を設定している

1. ミラーディスク用のパーティションを作成
   - Linuxの場合
     - /dev/oracleoci/oraclevdb1：RAW
     - /dev/oracleoci/oraclevdb2：ext4でフォーマット
   - Windowsの場合
     - D:\ ：ファイルシステム未作成
     - E:\ ：NTFSでフォーマット
1. CLUSTERPRO をインストールし、ライセンスを登録する
1. Cluster WebUIを起動し、設定モードからクラスタ生成ウィザードを実行する
1. 基本設定、インタコネクトを設定する
   - インタコネクト1
     - Node1：10.0.0.8
     - Node2：10.0.0.9
     - MDC：なし
   - インタコネクト2
     - Node1：10.0.10.8
     - Node2：10.0.10.9
     - MDC：mdc1
1. NP解決を設定する
   - 種類：Ping
   - ターゲット：10.0.0.1
1. フェイルオーバグループを作成する
1. ミラーディスクリソースを作成する
  - Linuxの場合
    - 詳細
      - ミラーパティションデバイス名：/dev/NMP1
      - マウントポイント：/mnt/md1
      - データパーティションデバイス名：/dev/oracleoci/oraclevdb2
      - クラスタパーティションデバイス名：/dev/oracleoci/oraclevdb1
      - ファイルシステム：ext4
  - Windowsの場合
    - 詳細
      - データパーティションのドライブ文字：E:\
      - クラスタパーティションのドライブ文字：D:\
      - ミラーディスクコネクト：mdc1
      - 起動可能サーバ：Node1, Node2
1. Azure プローブポートリソースを作成する
   - 詳細
     - プローブポート：26001
1. カスタムモニタリソース/カスタム監視リソースを作成する
   - 情報
     - 名前：genw1
   - 監視(固有) -> この製品で作成したスクリプト
      - Linuxの場合
        ```! /bin/sh
           /opt/nec/clusterpro/bin/clpazure_port_checker -h iaas.ap-tokyo-1.oraclecloud.com -p 443
           exit $?
        ```
      - Windowsの場合
        ```
            "C:\Program Files\EXPRESSCLUSTER\bin\clpazure_port_checker" -h iaas.ap-tokyo-1.oraclecloud.com -p 443
            EXIT %ERRORLEVEL%
        ```
   - 回復動作
     - 回復動作：最終動作のみ実行
     - 回復対象：LocalServer
     - 最終動作：何もしない
1. IP モニタリソース/IP 監視リソース(１つ目)を作成する
   - 情報
     - 名前：ipw1
   - 監視(共通)
     - 監視を行うサーバを選択する
       - 「独自に設定する」を選択して Node1 を追加
   - 監視(固有)
     - 「追加」を選択してNode2のIPアドレス(10.0.10.9)を追加
   - 回復動作
     - 回復動作：最終動作のみ実行
     - 回復対象：LocalServer
     - 最終動作：何もしない
1. IP モニタリソース/IP 監視リソース(２つ目)を作成する
   - 情報
     - 名前：ipw2
   - 監視(共通)
     - 監視を行うサーバを選択する
       - 「独自に設定する」を選択して Node2 を追加
   - 監視(固有)
     - 「追加」を選択してNode1のIPアドレス(10.0.10.8)を追加
   - 回復動作
     - 回復動作：最終動作のみ実行
     - 回復対象：LocalServer
     - 最終動作：何もしない
1. マルチターゲットモニタリソース/マルチターゲット監視リソースを作成する
   - 監視(固有)
     - genw1, ipw1, ipw2 を追加する
   - 回復動作
     - 回復動作：最終動作のみ実行
     - 回復対象：LocalServer
     - 最終動作前にスクリプトを実行する：チェックを入れる
     - 最終動作：何もしない
     - スクリプト設定 -> この製品で作成したスクリプト -> 編集 を選択
       - Linuxの場合
         ```#! /bin/sh
            /opt/nec/clusterpro/bin/clpazure_port_checker -h 127.0.0.1 -p 8080
            if [ $? -ne 0 ]
            then
            clpdown
            exit 0
            fi

            /opt/nec/clusterpro/bin/clpazure_port_checker -h <ロードバランサーのフロントエンドIP(パブリックIP アドレス)> -p 80
            if [ $? -ne 0 ]
            then
            clpdown
            exit 0
            fi
            ```
       - Windowsの場合
         ```
            rem ********************
            rem Check Active Node
            rem ********************
            "C:\Program Files\EXPRESSCLUSTER\bin\clpazure_port_checker" -h 127.0.0.1 -p 8080
            IF NOT "%ERRORLEVEL%" == "0" (
            GOTO CLUSTER_SHUTDOWN
            )
            rem ********************
            rem Check DNS
            rem ********************
            "C:\Program Files\EXPRESSCLUSTER\bin\clpazure_port_checker" -h <ロードバランサーのフロントエンドIP(パブリックIP アドレス)> -p 80
            IF "%ERRORLEVEL%" == "0" (
            GOTO EXIT
            )
            rem ********************
            rem Cluster Shutdown
            rem ********************
            :CLUSTER_SHUTDOWN
            clpdown
            rem ********************
            rem EXIT
            rem ********************
            :EXIT
            EXIT 0
            ```
     - タイムアウト：15秒
1. 以下のモニタ/監視リソースはフェイルオーバグループのリソース作成時に自動的に作成される
   - Linuxの場合
     - ミラーディスクコネクトモニタリソース
     - ミラーディスクモニタリソース
     - Azure プローブポートモニタリソース
     - Azure ロードバランスモニタリソース
   - Windowsの場合
     - ミラーコネクト監視リソース
     - ミラーディスク監視リソース
     - Azure プローブポート監視リソース
     - Azure ロードバランス監視リソース
1. クラスタのプロパティ -> タイムアウト
   - ハートビートタイムアウト：120秒  ※ A+B+Cよりも大きい値を設定
     - A ：ipw1,ipw2,mtw1 の[インターバル] × ([リトライ回数] + 1)
           ※3つのうち最も大きい値を選択
     - B ：マルチターゲット監視リソースの [インターバル] × ([リトライ回数] + 1)
     - C ：30秒
1. Cluster WebUI から設定の反映を行う

CLUSTERPRO の動作確認
---
1. 参考に記載した構築ガイドの「5.4 動作確認」を参照して動作確認を行う

参考
---
- CLUSTERPRO X 4.1 Microsoft Azure 向け HA クラスタ 構築ガイド (Linux 版)
   - https://jpn.nec.com/clusterpro/clpx/doc/guide/HOWTO_Azure_X41_Linux_JP_01.pdf
- CLUSTERPRO X 4.1 Microsoft Azure 向け HA クラスタ 構築ガイド (Windows 版)
   - https://jpn.nec.com/clusterpro/clpx/doc/guide/HOWTO_Azure_X41_Windows_JP_01.pdf
