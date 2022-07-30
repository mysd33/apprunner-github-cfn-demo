# SpringBoot APをGitHubとAppRunnerで動作させるCloudFormationサンプルテンプレート

## 構成
* システム構成図
![システム構成](img/apprunner-github.png)


* CI/CD、コンテナ起動はAppRunnerで実現
    * ソースコードをGitHubで管理し、git pushするとAppRunnerでビルド&デプロイ
        * AppRunnerがサポートするソースコードリポジトリは現状GitHubのみ
        * https://docs.aws.amazon.com/ja_jp/apprunner/latest/dg/architecture.html
* メトリックスのモニタリング
    * AppRunnerの機能で実現
        * https://docs.aws.amazon.com/ja_jp/apprunner/latest/dg/monitor-cw.html
* ログの転送
    * AppRunnerの機能でCloudWatch Logsへログ出力
        * https://docs.aws.amazon.com/ja_jp/apprunner/latest/dg/monitor-cwl.html
* オートスケーリング
    * AppRunnerの機能で実現
        * https://docs.aws.amazon.com/ja_jp/apprunner/latest/dg/manage-autoscaling.html
    * 現在、CloudFormationでは、AutoScalingConfigurationのリソースが作成不可なのでデフォルトのオートスケーリング設定
* VPCコネクタによるVPC内のリソースアクセス
    * VPCコネクタにより、パブリックでないRDS等のVPCリソースへのアクセスが可能
        * TODO: 本サンプルでは未実施
    * インバウンドトラフィックには利用できないので、インバウンドトラフィックは従来どおりパブリックなアクセスのみ、IP等でのアクセス制限もできないのでAppRunnerは完全閉域な構成には向かない

## 事前準備
* 別途、以下の2つのSpringBootAPのプロジェクトが以下のリポジトリ名でGitHubにある前提
  * backend-for-frontend
    * BFFのAP
    * backend-for-frontendという別のリポジトリに資材は格納
  * backend
    * BackendのAP
    * backendという別のリポジトリに資材は格納
* マネージドコンソール等、AppRunnerの「GitHub接続」を作成しておく
    * https://docs.aws.amazon.com/ja_jp/apprunner/latest/dg/manage-connections.html    

## AppRunner環境

### 1. AppRunnerの作成
* Backendアプリケーションの起動
```sh
aws cloudformation validate-template --template-body file://cfn-apprunner-backend.yaml

aws cloudformation create-stack --stack-name APPRUNNER-BACKEND-Stack --template-body file://cfn-apprunner-backend.yaml ParameterKey=GitHubConnectionArn,ParameterValue=(GitHub接続のARN)ParameterKey=GitHubRepositoryUrl,ParameterValue=(GitHubのリポジトリのURL) 
#例
aws cloudformation create-stack --stack-name APPRUNNER-BACKEND-Stack --template-body file://cfn-apprunner-backend.yaml --parameters ParameterKey=GitHubConnectionArn,ParameterValue=arn:aws:apprunner:ap-northeast-1:999999999999:connection/apprunner-example-connection/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX ParameterKey=GitHubRepositoryUrl,ParameterValue=https://github.com/xxxxx/backend
```

* BFFアプリケーションの起動
```sh
aws cloudformation validate-template --template-body file://cfn-apprunner-bff.yaml
aws cloudformation create-stack --stack-name APPRUNNER-BFF-Stack --template-body  file://cfn-apprunner-bff.yaml ParameterKey=GitHubConnectionArn,ParameterValue=(GitHub接続のARN)ParameterKey=GitHubRepositoryUrl,ParameterValue=(GitHubのリポジトリのURL) 

#例
aws cloudformation create-stack --stack-name APPRUNNER-BFF-Stack --template-body  file://cfn-apprunner-bff.yaml 
 --parameters ParameterKey=GitHubConnectionArn,ParameterValue=arn:aws:apprunner:ap-northeast-1:999999999999:connection/apprunner-example-connection/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX ParameterKey=GitHubRepositoryUrl,ParameterValue=https://github.com/xxxxx/backend-for-frontend
```

### 2. APの実行確認
* Backendアプリケーションの確認
  * ブラウザで「http://(AppRunnerのDNS名)/backend-for-frontend/index.html」を入力しフロントエンドAPの画面が表示される
    * CloudFormationの「APPRUNNER-BACKEND-Stack」スタックの出力「BackendServiceURI」のURLを参照

* BFFアプリケーションの確認    
  * ブラウザで「http://(AppRunnerのDNS名)/backend-for-frontend/index.html」を入力しフロントエンドAPの画面が表示される
    * CloudFormationの「APPRUNNER-BFF-Stack」スタックの出力「BffServiceURI」のURLを参照

### 3. CI/CDの確認
* ソースコードの変更
  * 何らかのソースコードの変更を加えて、GitHubにプッシュする
  * GitHubに新しいソースコードがプッシュされることで、AppRunnerでビルド実行され新しいAPがデプロイされることを確認する