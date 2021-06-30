# Amazon Interactive Video Service (IVS) と AWS Amplify を使って自分だけのオリジナル配信サイトを作る！
こんにちは！ [**株式会社スタートアップテクノロジー**](https://startup-technology.com/) 所属、  
そして先日、[**AWS Serverless HERO**](https://aws.amazon.com/developer/community/heroes/hidetoshi-matsui/)に選ばれました、[**JAWS-UG** 浜松支部](https://jawsug-hamamatsu.doorkeeper.jp/)の松井です！  

今回は **AWS** の各種マネージドサービスを活用した配信サイトの構築方法のお話をさせていただきます！  

オンライン配信しようとした場合、 **YouTube Live** などを活用する方法が考えられますね。  
しかし、既存サービスを活用する場合カスタマイズが難しく、何かと不自由もあるかと思います。  

そこで今回は、 **AWS** で提供されている **Amazon Interactive Video Service (IVS)** と **AWS Amplify** 、  
また **StreamYard** と言うライブストリームサービスを活用して、自分だけのオリジナル配信サイトを構築する方法をご紹介します！

## 必要要件

- 下記サービスの有効なアカウント
  - **AWS**
  - **GitHub**
- 数十円程度の **AWS** 利用料および **StreamYard** の最低 $20/month の利用料
- ストリーミングアプリケーションの **OBS** のインストール

## アーキテクチャ
- 今回のハンズオンで構築するアーキテクチャです。

![ivs-nuxt (1)](https://user-images.githubusercontent.com/38583473/122328796-3bffc400-cf6b-11eb-9d0b-bd146d2801aa.png)

- **Amplify Console** を使い、 **Nuxt.js** で作成した静的ページをデプロイ・ホスティングします。
- **OBS** という配信データを送信するアプリケーションを使って、 **Amazon IVS** に対してPC上から動画データを送信します。
- Webブラウザ側から、 **amazon-ivs-player** や、 **Video.js** といったライブラリを利用して、 **Amazon IVS** が提供する再生URLからストリーミングデータを取得して表示します。

***

- また、参考として、[**JAWS DAYS 2021 re:Connect**](https://jawsdays2021.jaws-ug.jp/) の配信サイトのアーキテクチャも紹介します。

![JAWS-DAYS-2021-STREAMING-V2 (5)](https://user-images.githubusercontent.com/38583473/121799787-fe4c2400-cc68-11eb-869f-4708aca5a4c3.png)

- **Amazon EventBridge** をトリガーにして　**AWS Lambda** を実行し、 **Amazon IVS** の配信視聴者数を取得し、 **Amazon DynamoDB** に保存する機能を実装しました。
- また、 **Amazon IVS** の **Timed Metadata** という機能を利用して、上記の視聴者数も含めたリアルタイムなデータのクライアントへの反映や、アンケート機能などを実装しました。
- **AWS Amplify** を利用して、配信に付随する各種データの管理ができる仕組みも構築しました。


***

## 目次

1. **IVS** の設定
2. **OBS** のインストール・設定
3. **Nuxt.js** プロジェクトのセットアップ
4. **Amplify Console** でデプロイ

## 1. **IVS** の設定

### **AWS マネジメントコンソール** にログイン
- [**AWS マネジメントコンソール**](https://console.aws.amazon.com/)にアクセスします。

***

<img width="1198" alt="スクリーンショット 2021-05-22 21 55 41" src="https://user-images.githubusercontent.com/38583473/119227442-db868e00-bb48-11eb-9d74-a43195f3dcf0.png">

- メールアドレスを入力して、**次へ**をクリックします。

***

<img width="1198" alt="スクリーンショット 2021-05-22 22 03 06" src="https://user-images.githubusercontent.com/38583473/119227950-6ff1f000-bb4b-11eb-9163-0534c30c4fc0.png">

- セキュリティチェックを入力して、**送信**をクリックします。

***

<img width="1198" alt="スクリーンショット 2021-05-22 22 18 01" src="https://user-images.githubusercontent.com/38583473/119228043-032b2580-bb4c-11eb-8de0-ff720c362ef1.png">

- パスワードを入力して、**サインイン**をクリックします。

***

### IVS チャネルを作成

<img width="887" alt="スクリーンショット 2021-05-22 22 30 10" src="https://user-images.githubusercontent.com/38583473/119228605-b137cf00-bb4e-11eb-879c-e062fac00f6c.png">

- **ivs** と入力して、 **Amazon Interactive Video Service** をクリックします。

***

<img width="887" alt="スクリーンショット 2021-05-22 22 44 26" src="https://user-images.githubusercontent.com/38583473/119228776-88640980-bb4f-11eb-8624-219168e7a167.png">

- **リージョンを 米国西部（オレゴン）に変更** をクリックします。

***

<img width="1125" alt="スクリーンショット 2021-05-22 22 51 13" src="https://user-images.githubusercontent.com/38583473/119228927-58693600-bb50-11eb-9cf3-3658f473391a.png">

- **チャネルの作成**をクリックします。

***

<img width="491" alt="スクリーンショット 2021-05-22 23 00 16" src="https://user-images.githubusercontent.com/38583473/119229203-c6fac380-bb51-11eb-9814-492a884a0f63.png">

- **チャネル名**に**ivs-nuxt-1**と入力し、**チャネルの作成**をクリックします。

***

<img width="1198" alt="スクリーンショット 2021-05-22 23 07 58" src="https://user-images.githubusercontent.com/38583473/119230716-69b64080-bb58-11eb-9eef-6e9d72495625.png">

- 作成した **ivs-nuxt-1** 詳細ページの、**ストリーム設定**の3点をコピーして控えます。
  - **取り込みサーバー**
  - **ストリームキー**
  - **再生URL**

***

## 2. **OBS** のインストール・設定(※Mac での方法のみご案内します)

### **OBS** のインストール
- [**OBS のインストールサイト**](https://obsproject.com/ja) へアクセスします。

***

<img width="1440" alt="スクリーンショット 2021-06-30 13 41 33" src="https://user-images.githubusercontent.com/38583473/123903254-509b7d80-d9a9-11eb-850f-4bd85929a6de.png">

- **Windows**, **Mac OS**, **Linux** の内、該当するものをクリックします（ダウンロードが始まります)。

***

<img width="1440" alt="スクリーンショット 2021-06-30 13 46 41" src="https://user-images.githubusercontent.com/38583473/123903584-ecc58480-d9a9-11eb-91dd-19f3759be4a6.png">

- ダウンロードが完了したら、画像の通りの箇所をクリックします。

***

<img width="697" alt="スクリーンショット 2021-06-30 13 29 47" src="https://user-images.githubusercontent.com/38583473/123903632-0797f900-d9aa-11eb-9965-3694bbf92b88.png">

- 画像のような表示になるので、左から右へドラッグ&ドロップします。

***

<img width="1440" alt="スクリーンショット 2021-06-30 13 50 38" src="https://user-images.githubusercontent.com/38583473/123904285-2fd42780-d9ab-11eb-983a-c7630f925d5d.png">

- アプリケーションとしてメニューから選択できるようになっていればOKです。

***

### **OBS** に **IVS** で作成したチャンネルを登録

<img width="1440" alt="スクリーンショット 2021-06-30 13 50 38のコピー" src="https://user-images.githubusercontent.com/38583473/123912371-ec33ea80-d9b7-11eb-9000-084004f4e4aa.png">

- **OBS** を起動します。

***

<img width="1079" alt="スクリーンショット 2021-06-30 15 30 14" src="https://user-images.githubusercontent.com/38583473/123914696-a298cf00-d9ba-11eb-95bd-c593643e42c8.png">

- 設定ウィザードが立ち上がるので、 **配信のために最適化し、録画は二次的なものとする** を選択して、 **次へ** をクリックする。

***

<img width="1079" alt="スクリーンショット 2021-06-30 15 31 58" src="https://user-images.githubusercontent.com/38583473/123914915-e55aa700-d9ba-11eb-9999-122162b0ba4f.png">

- 次の画面では、そのまま **次へ** をクリックします。

***

<img width="1079" alt="スクリーンショット 2021-06-30 15 41 05" src="https://user-images.githubusercontent.com/38583473/123915056-0d4a0a80-d9bb-11eb-80a3-eee44a715682.png">

- **1. IVS の設定** で控えた、下記の情報をそれぞれのフォームに入力し、 **次へ** をクリックします。
  - **サーバー**: **IVS** の、 rtmps://6468fb1f4abe.global-contribute.live-video.net:443/app/
  - **ストリームキー**: **IVS** の、 ストリームキー

***

<img width="1079" alt="スクリーンショット 2021-06-30 15 44 16" src="https://user-images.githubusercontent.com/38583473/123916738-f3a9c280-d9bc-11eb-8329-8ff409866d6f.png">

- **はい** をクリックします。
- テストが実行されるので、完了するまで待ちます（数分程度)。

***

<img width="1079" alt="スクリーンショット 2021-06-30 15 46 56" src="https://user-images.githubusercontent.com/38583473/123916809-07552900-d9bd-11eb-9710-7f056e0818a9.png">

- テストが完了したら、 **設定を適用** をクリックします。
- 引き続き **OBS** を使っていくので、このままアプリケーションを開いた状態にしておきます。

***

## 3. **Nuxt.js** プロジェクトのセットアップ

- 今回ページを公開するための **Nuxt.js** のフロントエンドプロジェクトをセットアップします。
- ハンズオンの中では具体的にコードの変更はしませんが、これ以降はこちらの環境で自由にカスタムしていただけます！

### **AWS Cloud9** 環境の構築

- [**こちら**](https://aws.amazon.com/jp/builders-flash/202008/amplify-crud-app#002)を参考に、 **AWS Cloud9** 環境を構築します。
  -  **Cloud9 の環境が整いました。** の箇所まで進めます。

### **Nuxt.js** プロジェクトのフォーク、クローン、セットアップ(これ以降は **Cloud9** のターミナルで操作してください)

- [**こちら**](https://github.com/matsuihidetoshi/ivs-nuxt)のリポジトリにアクセスします。

***

<img width="1437" alt="スクリーンショット 2021-06-12 23 36 44" src="https://user-images.githubusercontent.com/38583473/121780208-42441800-cbda-11eb-8cc2-25e871d87495.png">

- **Fork** をクリックします。

***

- **GitHub** からコードをクローンして、プロジェクトディレクトリに移動します。
- ※これ以降、**プロジェクトリポジトリに移動しないとうまくコマンドが実行できません**のでお気をつけください。

```
git clone https://github.com/[自分のGitHubアカウントのキー]/ivs-nuxt.git
cd ivs-nuxt
```

***

- **npm** パッケージをインストールします。

```
npm i
```

***

- テスト起動します。

```
npm run dev
```

***

<img width="1439" alt="スクリーンショット 2021-06-12 22 17 22" src="https://user-images.githubusercontent.com/38583473/121777405-d0b19d00-cbcc-11eb-8ad0-72088d315907.png">

- **Preview** をクリックしてから、 **Preview Running Application** をクリックします。

***

<img width="1439" alt="スクリーンショット 2021-06-12 22 28 07" src="https://user-images.githubusercontent.com/38583473/121777874-369f2400-cbcf-11eb-8f9a-5f731805abba.png">

- デモ用のストリーミング動画が表示されるのを確認します。

***

### **OBS** で配信開始

***

<img width="1079" alt="スクリーンショット 2021-06-30 16 15 52" src="https://user-images.githubusercontent.com/38583473/123919359-d4606480-d9bf-11eb-8c8e-2f09d7639bee.png">

- ストリーミングデータのソースを追加するために、画像の **+** をクリックします。

***

<img width="1079" alt="スクリーンショット 2021-06-30 16 16 30" src="https://user-images.githubusercontent.com/38583473/123919454-e8a46180-d9bf-11eb-8ff9-44d19865899f.png">

- **映像キャプチャデバイス** を選択します。

***

<img width="1079" alt="スクリーンショット 2021-06-30 16 18 25" src="https://user-images.githubusercontent.com/38583473/123920216-aaf40880-d9c0-11eb-9553-79c8958ea777.png">

- デフォルトのまま、 **OK** をクリックします。

***

<img width="1079" alt="スクリーンショット 2021-06-30 16 19 21" src="https://user-images.githubusercontent.com/38583473/123921043-906e5f00-d9c1-11eb-9083-9ff0df186d00.png">

- **デバイス** に、各自の環境の有効なカメラデバイス(例では **Snap Camera**, **Mac** のデフォルトの場合、 **FaceTime HDカメラ (内臓)**)を選択し、 **OK** をクリックします。

***

<img width="1079" alt="スクリーンショット 2021-06-30 16 20 50" src="https://user-images.githubusercontent.com/38583473/123921625-2b673900-d9c2-11eb-8716-176a1478a310.png">

- ウィンドウサイズをドラッグ&ドロップで合わせます。

***

<img width="1079" alt="スクリーンショット 2021-06-30 16 21 28" src="https://user-images.githubusercontent.com/38583473/123921663-37eb9180-d9c2-11eb-9106-727d8a43e087.png">

- **配信開始** をクリックします。

***

### **Nuxt.js** プロジェクトに自分の **再生URL** を設定

- 一度、起動している開発環境のサーバーを停止します(ターミナルで `ctrl + c`)。

```
ctrl + c
```

***

- ターミナルで下記を実行し、[こちら](https://github.com/matsuihidetoshi/ivs-nuxt/blob/main/README.md#ivs-%E3%83%81%E3%83%A3%E3%83%8D%E3%83%AB%E3%82%92%E4%BD%9C%E6%88%90)で控えた自分の **再生URL** を環境変数に設定します。

```
export IVS_NUXT_STREAM_URL="自分の再生URL"
```

***

- サーバーを再起動します。

```
npm run dev
```

***

- **Preview** しているページを再読み込みして、自分の配信が再生されることを確認します。

***

## 4. **Amplify Console** でデプロイ
- ここまでローカルでプロジェクトを進めてきましたが、いよいよ配信サイトを公開します！
- **AWS Amplify Console** から簡単なステップで公開できますので、その手順を解説します。

***

<img width="1437" alt="スクリーンショット 2021-06-13 0 18 08" src="https://user-images.githubusercontent.com/38583473/121829300-18cfdd00-ccfd-11eb-9809-907575039917.png">

- **AWS マネジメントコンソール**で検索フォームに`amplify`と入力します。
- **AWS Amplify** が表示されたら、クリックします。

***

<img width="1437" alt="スクリーンショット 2021-06-13 0 21 00" src="https://user-images.githubusercontent.com/38583473/121829356-39983280-ccfd-11eb-9c67-776b262e3cd3.png">

- **GET STARTED** をクリックします。

***

<img width="1437" alt="スクリーンショット 2021-06-13 0 21 59" src="https://user-images.githubusercontent.com/38583473/121829380-4ae13f00-ccfd-11eb-9ec6-becaa7463baa.png">

- **Get started** の下の右側のセクションの **Get started** ボタンをクリックします。

***

<img width="1093" alt="スクリーンショット 2021-06-13 0 23 27" src="https://user-images.githubusercontent.com/38583473/121829415-6b10fe00-ccfd-11eb-8439-51044c50c1b0.png">

- **GitHub** を選択し、 **Continue** をクリックします。

***

<img width="588" alt="スクリーンショット 2021-06-13 0 27 25" src="https://user-images.githubusercontent.com/38583473/121829474-985dac00-ccfd-11eb-9350-35712844845e.png">

- **Authorize aws-amplify-console** をクリックします。

***

<img width="799" alt="スクリーンショット 2021-06-13 0 28 40" src="https://user-images.githubusercontent.com/38583473/121829842-9cd69480-ccfe-11eb-9007-41e171884fd2.png">

- **リポジトリ**からForkしたご自身の **ivs-nuxt** リポジトリを選択します。
- **main** ブランチを選択して、 **次へ** をクリックします。

***

![スクリーンショット 2021-06-14 15 48 52](https://user-images.githubusercontent.com/38583473/121999549-69703480-cde8-11eb-9c24-d5eb48d69aff.png)

- **ビルド設定の追加**の **Edit** をクリックして、後述の `amplify.yml` 通り変更してください。
- **Advanced settings** をクリックします。
- **環境変数**の **Key** に `IVS_NUXT_STREAM_URL` を、 **Value** に先ほど[こちら](https://github.com/matsuihidetoshi/ivs-nuxt#ivs-%E3%83%81%E3%83%A3%E3%83%8D%E3%83%AB%E3%82%92%E4%BD%9C%E6%88%90)で控えた **再生URL** を入力します。
- 次に、**次へ**をクリックしてください。

**amplify.yml**
```
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm install
    build:
      commands:
        - npm run build
        - npm run generate
  artifacts:
    # IMPORTANT - Please verify your build output directory
    baseDirectory: dist
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

***

<img width="792" alt="スクリーンショット 2021-06-13 19 08 23" src="https://user-images.githubusercontent.com/38583473/121830059-24bc9e80-ccff-11eb-9e63-c902ff769dd9.png">

- **保存してデプロイ**をクリックします。

***

<img width="1122" alt="スクリーンショット 2021-06-13 19 12 38" src="https://user-images.githubusercontent.com/38583473/121830087-369e4180-ccff-11eb-91e4-7f7140f51a23.png">

- プログレスバーのステータスが**検証**になるのを待ちます。
- その後、**ドメイン**の箇所に記載されているURLをクリックしてください。
- 配信ページが **Cloud9** で実行したプレビューページ同様表示されればOKです！

***

## まとめ
いかがでしたでしょうか？  
こちらの配信サイトは **JavaScript(Vue.js/Nuxt.js)** ベースでできておりますので、  
もちろんこのリポジトリからカスタムしていただいて独自の機能を追加したり、思い思いのデザインにしていただくことが可能です！  
例えば、カスタマイズの方向性としては下記などが考えられます。

- **Amazon IVS** の **Timed Metadata** を利用して、バックエンドとのインタラクションを活用した機能を構築する。
- **Amazon IVS** の録画データの保存機能を有効化して、 **Amazon S3** に配信動画を保存する。

ぜひ、色々とカスタムして遊んでみてください！！  
**Happy coding!**

