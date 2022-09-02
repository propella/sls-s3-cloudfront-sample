Serverless Framework + S3 + CloudFront
======================================

https://qiita.com/propella/items/9a16f51ea4d33b5b8e3d

たまに静的な HTML をちょこっと外部に公開したい時があります。そんな時に AWS S3 + CloudFront は便利なのですが、AWS Console を触るのは嫌なので Serverless Framework を使う方法を試しました。Serverless Framework を単に CloudFormation + アップローダとして使っています。

例えば `out/index.html` というファイルがあるとします。これを今から公開します。

```html
<h1>Hello, World!</h1>
````

まず必要なツールをインストールしたり、aws の権限を設定したりします。ここでは serverless コマンドだけグローバルで、プラグインをローカルにインストールしました。

```shell
npm install -g serverless
npm install -D serverless-s3-sync serverless-cloudfront-invalidate
export AWS_PROFILE=hoge
```

`serverless.yml` というファイルを用意します。

```yaml
service: sls-s3-cloudfront-sample
frameworkVersion: "3"

plugins:
  - serverless-s3-sync
  - serverless-cloudfront-invalidate

provider:
  name: aws
  region: ap-northeast-1

custom:
  s3Sync:
    - bucketName: ${self:service}-${sls:stage}
      localDir: out/
      acl: public-read
  cloudfrontInvalidate:
    - distributionIdKey: DistributionId
      items:
        - "/*"

resources:
  Resources:
    Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead
        BucketName: ${self:service}-${sls:stage}
        WebsiteConfiguration:
          IndexDocument: index.html
    Distribution:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          DefaultCacheBehavior:
            TargetOriginId: ${self:service}-${sls:stage}
            ForwardedValues:
              QueryString: false
            ViewerProtocolPolicy: https-only
          Enabled: true
          DefaultRootObject: index.html
          Origins:
            - CustomOriginConfig:
                OriginProtocolPolicy: http-only
              DomainName: ${self:service}-${sls:stage}.s3-website-${aws:region}.amazonaws.com
              Id: ${self:service}-${sls:stage}
  Outputs:
    URL:
      Value: !Sub "https://${Distribution.DomainName}"
    DistributionId:
      Value: !Ref Distribution

```

デプロイします。

```
serverless deploy --verbose

....
  URL: https://d1dwqegtyup9lu.cloudfront.net ← このような物が表示されたら成功。
```

要所要所解説します。

```yaml
custom:
  s3Sync:
    - bucketName: ${self:service}-${sls:stage}
      localDir: out/
      acl: public-read
```

ここでは `serverless-s3-sync` というプラグインの設定をしています。`localDir:` でアップロードしたいディレクトリを指定します。`acl: public-read` で外部に公開します。

```yaml
  cloudfrontInvalidate:
    - distributionIdKey: DistributionId
      items:
        - "/*"
```

`serverless-cloudfront-invalidate` プラグインの設定です。これによりデプロイするたびに CloudFront のキャッシュが消えます。 `distributionIdKey: DistributionId` というのが非常にわかりにくいのですが、ここで操作する CloudFront Distribution の ID を間接的に指定します。デプロイの事前に ID はわからないので、後述の CloudFormation の Output セクションで Distribution ID を出力して、その Logical ID を distributionIdKey として指定できます。

消したいキャッシュは items に指定します。ここでは "/*" なので全て消しています。

```yaml
resources:
  Resources:
    Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead
        BucketName: ${self:service}-${sls:stage}
        WebsiteConfiguration:
          IndexDocument: index.html
```

S3 を作成してウェブサイトとして公開します。多くの場合 S3 だけで十分だと思います。加えて CloudFront が必要なのは次の事情がある時です。

* HTTP ではなく HTTPS を使いたい。
  * HTTPS が欲しいだけなら S3 の Object URL で実現可能です。ただし IndexDocument 機能がありません。Object URL はいくつか種類があります。
    * DomainName: https://(bucket).s3.amazonaws.com/index.html
    * DualStackDomainName (ipv6): https://(bucket).s3.dualstack.(region).amazonaws.com/index.html
    * RegionalDomainName: https://(bucket).s3.(region).amazonaws.com/index.html
* `なんとか/` にアクセスした時に `なんとか/index.html` が出て欲しい。(IndexDocument 機能)
  * IndexDocument 機能が欲しいだけなら S3 の Static website hosting で実現可能です。ただし HTTPS が動きません。
  * URL は http://(bucket).s3-website-(region).amazonaws.com です。

もしこれらに当てはまる場合だけ以下の CloudFront が必要になります。

```yaml
    Distribution:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Origins:
            - CustomOriginConfig:
                OriginProtocolPolicy: http-only
              DomainName: ${self:service}-${sls:stage}.s3-website-${aws:region}.amazonaws.com
              Id: ${self:service}-${sls:stage}
          DefaultCacheBehavior:
            TargetOriginId: ${self:service}-${sls:stage}
            ForwardedValues:
              QueryString: false
            ViewerProtocolPolicy: https-only
          Enabled: true
          DefaultRootObject: index.html
```

CloudFront の設定はこのようになります。ややこしい部分を解説します。

* まず、`Origins` でコンテンツ本体を指定します。
  * ここで、`DomainName` に S3 Static website hosting のドメインを指定するのがコツです。Object URL を指定してしまうと `なんとか/` から `なんとか/index.html` への変換ができません。もっというとルートだけなら `DefaultRootObject: index.html` の指定で変換可能なのですが、サブディレクトリを変換してくれません。
  * `Id: ${self:service}-${sls:stage}` は何でも良いのですが `DefaultCacheBehavior` で使います。
* `DefaultCacheBehavior` でキャッシュの動作を指定します。
  * `TargetOriginId` は `Origins` で指定した `Id` です。
  * `ForwardedValues` は不推奨ですが、指定が簡単なので使いました。`QueryString: false` によってクエリ文字列が異なっても同じコンテンツとみなしてキャッシュします。

```yaml
  Outputs:
    URL:
      Value: !Sub "https://${Distribution.DomainName}"
    DistributionId:
      Value: !Ref Distribution
```

このような Output セクションがあると、デプロイ完了後 URL をすぐに知る事ができて便利です。`DistributionId` の方は `serverless-cloudfront-invalidate` プラグインの `distributionIdKey` で必要な項目です。

完成品: https://github.com/propella/sls-s3-cloudfront-sample

# 参考

* [Serverless Framework で SPA 環境を構築してみる（with Lambda@Edge & API Gateway） - Qiita](https://qiita.com/wind-up-bird/items/d8a43f61b9f81d79185d)
* [Static Websites On AWS S3 With Serverless Framework](https://www.serverlessops.io/blog/static-websites-on-aws-s3-with-serverless-framework)
* [Setting Up Serverless Framework With AWS](https://www.serverless.com/framework/docs/getting-started)
* [serverless-cloudfront-invalidate - npm](https://www.npmjs.com/package/serverless-cloudfront-invalidate)
* [AWS::CloudFront::Distribution - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-distribution.html)
