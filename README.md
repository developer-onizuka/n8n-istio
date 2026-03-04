# n8n環境を、AWS WorkSpaces Webで実現する

# 1. ゴール
今回は、AWS環境を使って、Kubernetes上にAIエージェントによるSlackチャットボットを構築することを目指します。この際、Service Meshとしてistioを使い、HTTPS環境を構築することでセキュアな実行環境を構築することにします。<br><br>

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
| **Networking** | **AWS Load Balancer (NLB)** | 外部からのアクセスエンドポイント | パブリック |

<br><br>

そこで、WorkSpacesを介したセキュアなデスクトップ接続により、外部公開のリスクを排除し、厳格なガバナンスとコンプライアンスを実現します。WorkSpaces上のセキュアなデスクトップ環境で操作を完結させることで、ローカルPCへのデータ保存を制限し、情報漏洩リスクを最小化することも狙いです。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-aws2.drawio.png" width="720">

### 改良後のコンポーネント構成
| カテゴリ | サービス | 役割 | サブネット |
| :--- | :--- | :--- | :--- |
| **Compute** | **Amazon EC2** | Istio Service Mesh や AIエージェント関連サービスのホスト | プライベート |
| **Networking** | **AWS Load Balancer (NLB)** | Amazon Workspacesからのアクセスエンドポイント | プライベート |
| **Directory Service** | **Simple AD or AWS Managed Microsoft AD** | Amazon Workspacesのユーザー認証基盤 | プライベート |
| **User Access** | **Amazon WorkSpaces** | AIエージェント環境へアクセスするためのVDI環境 | プライベート |

---

## 3. 外部アクセスの方法論 (Identity-Aware Proxy)

オンプレミス環境では MetalLB を用いた外部公開が一般的ですが、本構成では AWS のマネージドサービスを最大限活用し、AWS WorkSpaces + NLB + Istio による強固な閉域アクセス環境を実現します。

ユーザー認証には Simple AD と連携した WorkSpaces を採用し、物理的に分離されたデスクトップ環境からのアクセスのみを許可。ネットワーク階層では NLB (Network Load Balancer) によるプロトコルパススルーを行い、最終的な HTTPS 終端を Istio Ingress Gateway で実施することで、証明書管理の柔軟性とゼロトラストなトラフィック制御を両立させています。

### 認証・アクセスフロー

本構成では、インターネットにエンドポイントを公開せず、**AWS WorkSpaces** という制御されたデスクトップ環境からのアクセスのみを許可する「閉域アクセスモデル」を採用しています。

1. **Access Control (VDI Auth):** ユーザーは **Simple AD** で認証された **AWS WorkSpaces** にログインします。このディレクトリサービスによる認証が、システムへの第一の関門となります。

2. **Request (Internal DNS):** WorkSpaces 上のセキュアなブラウザから、内部専用ドメイン（例: `n8n.internal`）を使用してアクセスします。リクエストは VPC 内の **Route 53 Private Hosted Zone** を介して NLB へ解決されます。

3. **Transport (L4 Passthrough):** **NLB (Network Load Balancer)** がリクエストを受信。L4（TCP）レイヤーで動作するため、HTTPS 暗号化パケットを書き換えることなく、そのまま **Istio Ingress Gateway** へパススルー転送します。

4. **Encryption (TLS Termination):** **Istio Ingress Gateway** が、Kubernetes Secret 内に保持された証明書を用いて HTTPS を終端（SSL復号）します。これにより、証明書管理の柔軟性を Kubernetes 側に集約しています。

5. **Verify & Route:** Istio が `AuthorizationPolicy` を適用。アクセス元が WorkSpaces のサブネット CIDR であることを最終検証し、正規のトラフィックのみを n8n Pod へルーティングします。



---

### この構成のメリット

| 項目 | 特徴 |
| :--- | :--- |
| **ゼロトラスト** | 「認証されたデバイス（WorkSpaces）」からのアクセスのみを信頼する構造。 |
| **完全閉域化** | パブリックなインターネットからの攻撃パス（外部IP）を物理的に排除。 |
| **構成の簡素化** | 通常、ALB で外部公開する際は n8n 自体のログイン画面が露出するリスクを避けるため、前段での Cognito 連携（IDベースの認証）が強く推奨されるが、本構成では WorkSpaces (Simple AD) で入口を固めることで、ALB 側の複雑な認証・証明書設定を排し、Istio 側で暗号化を完結させているのが特徴です。 |

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





