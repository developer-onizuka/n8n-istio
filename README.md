# n8n環境を、AWS Cognito + AWS WorkSpaces Webを実現する

# 1. ゴール
今回は、AWS環境を使って、Kubernetes上にAIエージェントによるSlackチャットボットを構築することを目指します。この際、Service Meshを使い、HTTPS環境を構築することでセキュアな実行環境を構築することにします。<br><br>

### 1-1. AIエージェントの完成イメージ
- 左半分：任意のpdfをアップロードし、ベクトルデータベース化するフロー<br>
- 右半分：Slackのチャネルに投稿があった場合、それをトリガーにして投稿内容をLLMが理解し、保存されているナレッジを使用して、ユーザーからの質問にSlackで回答するフロー<br><br>
- Left half: A flow for uploading any PDF and creating a vector database.<br>
- Right half: When a post is made to a Slack channel, it triggers LLM to understand the post and use stored knowledge to answer user questions via Slack.<br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-slack-RAG.png" width="960">

### 1-2. Slackチャットボットの受け答えイメージ
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/slack-bot.png" width="960">

## 2. アーキテクチャ概要

実装イメージを以下に示します。n8nはプライベートサブネットで構成してはいますが、n8nのサービスが外部から参照可能になってしまうことが課題です。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-aws.drawio.png" width="720">

### コンポーネント構成
| カテゴリ | サービス | 役割 | サブネット |
| :--- | :--- | :--- | :--- |
| **Compute** | **Amazon EC2** | Istio Service Mesh や AIエージェント関連サービスのホスト | プライベート |
| **Networking** | **AWS Load Balancer (ALB)** | 外部からのアクセスエンドポイント | パブリック |

そこで、WorkSpacesを介したセキュアなデスクトップ接続により、外部公開のリスクを排除し、厳格なガバナンスとコンプライアンスを実現します。WorkSpaces上のセキュアなデスクトップ環境で操作を完結させることで、ローカルPCへのデータ保存を制限し、情報漏洩リスクを最小化することも狙いです。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-aws2.drawio.png" width="720">

### 改良後のコンポーネント構成
| カテゴリ | サービス | 役割 | サブネット |
| :--- | :--- | :--- | :--- |
| **Compute** | **Amazon EC2** | Istio Service Mesh や AIエージェント関連サービスのホスト | プライベート |
| **Networking** | **AWS Load Balancer (ALB)** | 内部からのアクセスエンドポイント | プライベート |
| **Directory Service** | **Simple AD or AWS Managed Microsoft AD** | Amazon Workspacesのユーザー認証基盤 | プライベート |
| **User Access** | **Amazon WorkSpaces** | AIエージェント環境へアクセスするためのVDI環境 | プライベート |

---

## 3. 外部アクセスの方法論 (Identity-Aware Proxy)

オンプレミス環境ではMetalLBを使うことが多いのですが、今回はn8nの実行環境を、AWSの **ALB + Cognito + Istio** による多層認証環境で実現します。

### 認証フロー
1. **Request:** ユーザーが `n8n.example.com` にアクセス。
2. **Authenticate:** ALBが **Amazon Cognito** へリダイレクト。ユーザーはID/PWまたはSAML/OIDCでログイン。
3. **Authorize:** 認証成功後、ALBがリクエストヘッダーに **JWT (JSON Web Token)** を付与し、Istio Ingress Gatewayへ転送。
4. **Enforce:** Istioが `RequestAuthentication` ポリシーを用いてJWTの署名を検証し、正規のアクセスのみをn8n Podへルーティング。

---

## 3. 管理アクセスの方法論 (WorkSpaces Web)

運用管理（n8nの環境設定、認証情報の更新等）は、インターネットに公開されたURL経由ではなく、**Amazon WorkSpaces Web** を介して行います。

* **ブラウザ分離:** 管理者のPCにはデータを残さず、VPC内のセキュアなブラウザからのみ管理画面へアクセス。
* **閉域接続:** n8nのエディタ画面へのアクセスを、WorkSpaces Webが所属するサブネットのIPアドレスからのみに制限（Security Group / Istio AuthorizationPolicy）。

---

## 4. 主な設定ファイル (YAML)

### A. AWS Load Balancer Controller (Ingress)
ALBでCognito認証を強制するためのアノテーション設定です。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: n8n-ingress
  namespace: n8n
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    # Cognito 連携設定
    alb.ingress.kubernetes.io/auth-type: cognito
    alb.ingress.kubernetes.io/auth-idp-cognito: '{"UserPoolArn":"arn:aws:cognito-idp:ap-northeast-1:123456789012:userpool/ap-northeast-1_xxxx","UserPoolClientId":"xxxx","UserPoolDomain":"your-domain"}'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-1:123456789012:certificate/uuid
spec:
  rules:
    - host: n8n.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: istio-ingressgateway
                port:
                  number: 80





