## Azure App Service に SlackBot をデプロイしようの巻

### 前提
- Azureアカウント作成済み
- Azure にサブスクリプション作成済み
- SlackにBot を追加済み
- SlackBotToken を取得済み
- SlackBot は JBot (Java) でのサンプルを使用
- アプリは Git に登録済み
- 実行環境は mac（Windows は適宜読み替えること）

---

### 準備
1. Azure CLI のインストールとサインイン  
https://docs.microsoft.com/ja-jp/cli/azure/install-azure-cli-macos?view=azure-cli-latest
```
  $Azure CLI インストール
  brew update && brew install azure-cli

  $Azure CLI ログイン
  az login
```

2. Azure App Service on Linux の作成  
https://docs.microsoft.com/en-us/cli/azure/appservice/ase?view=azure-cli-latest#az-appservice-ase-create  
| name | value |
| --- | --- |
| サブスクリプション | EX: *Azure サブスクリプション 1* |
| リソースグループ | EX: *sibot-rg* |
| アプリ名 | EX: *sibot-kun*　※AzureApp全体で一意の名称 |
| OS | EX: *linux* |
| ランタイムスタック | *Java SE* |
| SKUとサイズ | *F1*（無料） |

  ```
  export subsc=Azure サブスクリプション 1
  export rgroup=sibot-rg
  export appname=sibot-kun

  $優先サブスクリプションを設定する
  az account set --subscription '$subsc'

  $リソースグループを作成する
  az group create -g $rgroup --location japaneast
```
```
  <Deploy in Linux>
  Freeプランのlinux環境は１つのサブスクリプションに１つまでなので注意

  $Azure App Service Plan を作成する
  az appservice plan create --name $appname --resource-group $rgroup --sku F1 --is-linux

  $Azure App Service Web を作成する
  az webapp create --name $appname --resource-group $rgroup --plan $appname --runtime "JAVA|8-jre8"
  ```
  ```
  <Deploy in Windows>
  Windows にデプロイする場合は、恐らくこのコマンドで可能。

  $Azure App Service Plan を作成する
  az appservice plan create --name $appname --resource-group $rgroup --sku F1

  $Azure App Service Web を作成する
  az webapp create --name $appname --resource-group $rgroup --plan $appname --runtime "java|1.8|Java SE|8"
  ```

### プロジェクトの作成
1. サンプルプロジェクトのクローン  
https://docs.microsoft.com/ja-jp/azure/java/spring-framework/deploy-spring-boot-java-app-with-maven-plugin  
gitに構成ずみのプロジェクトがある場合は、clone対象を適宜読み替える。  
```
  $ローカルにデプロイ作業用ディレクトリ作成  
  mkdir ~/CustomBotDeploy
  cd ~/CustomBotDeploy

  $クローン  
  git clone https://github.com/rampatra/jbot.git

  $プロジェクト直下に移動  
  cd jbot
```

2. *application.properties* の設定の変更  
`vi jbot-example/src/main/resources/application.properties`  
~~~
# Facebookと連携しないので以下に変更
spring.profiles.active=slack
# SlackBotのAPIトークンに変更
slackBotToken=xoxb-xxxxxxxxxxxxxxxxxx
~~~
  *※xoxb-xxxxxxxxxxxxxxxxxx = Slackに作成したBotのAPIトークン*

3. 依存関係を追加
Javaバージョンが9以上だとエラ〜が出るので、pomの <dependencies> にjaxbの依存関係を追加する  
`vi jbot-example/pom.xml`    
```
  <dependency>
      <groupId>javax.xml.bind</groupId>
      <artifactId>jaxb-api</artifactId>
      <version>2.1</version>
  </dependency>
```

4. ビルド  
```
$maven コマンドでビルド
mvn clean install
cd jbot-example
mvn clean package
```

5. 起動確認  
起動を確認  
`mvn spring-boot:run`  
停止する  
`control+c`

### いざデプロイ！
1. プラグインの変更
azureにデプロイできる様に jbot-example/pom.xml ファイルのプラグインを追加する  
`spring-boot-maven-plugin` の下に追加
~~~
  <plugin>
        <groupId>com.microsoft.azure</groupId>
        <artifactId>azure-webapp-maven-plugin</artifactId>
        <version>1.9.0</version>
  </plugin>
~~~

2. 設定を構成する(jbot-example/pom.xml の更新)  
構成コマンドを実行する  
`mvn azure-webapp:config`

  質問に答えて完了。Windowsにデプロイする場合は適宜読み替えて。
```
  Define value for OS(Default: Linux):
    1. linux [*]
    2. windows
    3. docker
    Enter index to use: 1
  Define value for javaVersion(Default: Java 8):
    1. Java 11
    2. Java 8 [*]
    Enter index to use: 2

  Please confirm webapp properties
  AppName : sibot-kun
  ResourceGroup : sibot-rg
  Region : japaneast
  PricingTier : F1
  OS : Linux
  RuntimeStack : JAVA 8-jre8
  Deploy to slot : false
  Confirm (Y/N)? : Y
```
  ※設定が違う箇所があれば「Y」にしてから再度コマンド実行して修正する  
```
  Please choose which part to config
    1. Application
    2. Runtime
    3. DeploymentSlot
    Enter index to use: 1 or 2 or 3
  ・・・
```

3. デプロイ先を確認する  
jbot-example/pom.xml の `<configuration>` セクションの設定を確認する  
異なるところは修正する  
  ***F1（無料）*** *になっているか確認するのすごく大事！*
```
<schemaVersion>V2</schemaVersion>
<resourceGroup>sibot-rg</resourceGroup>
<appName>sibot-kun</appName>
<pricingTier>F1</pricingTier>
<region>japaneast</region>
<runtime>
      <os>linux</os>
      <javaVersion>jre8</javaVersion>
      <webContainer>jre8</webContainer>
</runtime>
```

4. ポートを修正する  
80 のポートでリッスンするように jbot-example/pom.xml を修正する  
`<azure-webapp-maven-plugin>` の `<configuration>` セクションに `<appSettings>` セクションを追加する
```
  <appSettings>
      <property>
          <name>JAVA_OPTS</name>
          <value>-Dserver.port=80</value>
      </property>
  </appSettings>
```

6. ビルド＆デプロイ
```
  リビルド
  mvn clean package

  デプロイ
  mvn azure-webapp:deploy
```

7. Javaコマンドからデプロイ先のアプリを実行する（Windows にデプロイした場合のみ）
  1. Azureコンソールにブラウザでログインする  
  2. ホーム > リソースグループ > アプリ名 と選んでAppServiceの左メニューを表示する
  3. 左メニュー「開発ツール」の「コンソール」を選択する
  4. コンソール内で `java -jar app.jar` コマンドを実行する

完了です🎉  
デプロイに成功しても、Slackでオンラインになるまで**10分**くらいかかります。  
App Service を停止するとBOTは即座に停止しますが、Slack上では**5分程度**オンラインのままです。  
（辛抱強く待ちましょう）

---

### 〜おまけ〜
- SSHでデプロイ先(Azure App Service)に接続する方法(デプロイ先がlinuxの場合のみ)

 - ssh用の設定を取得する  
`az webapp create-remote-connection --subscription <subscription-id> --resource-group <resource-group-name> -n <app-name> &`  

   EX:  
`az webapp create-remote-connection --subscription 'Azure サブスクリプション 1' --resource-group 'sibot-rg' -n 'sibot-kun' &`
  > Opening tunnel on port: 57789  
  > SSH is available { username: root, password: Docker! }

 - sshする  
`ssh root@127.0.0.1 -p 57789`  
`password Docker!`

---
