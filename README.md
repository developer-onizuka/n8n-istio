# n8n環境を、AWS WorkSpaces Webで実現する

# 1. ゴール
今回は、AWS環境を使って、Kubernetes上にAIエージェントによるSlackチャットボットを構築することを目指します。この際、Service Meshとしてistioを使い、HTTPS環境を構築することでセキュアな実行環境を構築することにします。WorkSpaces上のセキュアなデスクトップ環境で操作を完結させることで、ローカルPCへのデータ保存を制限し、情報漏洩リスクを最小化する<br><br>

### 1-1. AIエージェントの完成イメージ
- 左半分：任意のpdfをアップロードし、ベクトルデータベース化するフロー<br>
- 右半分：Slackのチャネルに投稿があった場合、それをトリガーにして投稿内容をLLMが理解し、保存されているナレッジを使用して、ユーザーからの質問にSlackで回答するフロー<br><br>
- Left half: A flow for uploading any PDF and creating a vector database.<br>
- Right half: When a post is made to a Slack channel, it triggers LLM to understand the post and use stored knowledge to answer user questions via Slack.<br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-slack-RAG.png" width="960">

### 1-2. Slackチャットボットの受け答えイメージ
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/slack-bot.png" width="960">

## 2. アーキテクチャ概要

誰もが思いつくであろう一般的な実装イメージを以下に示します。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-aws.drawio.png" width="720">

| カテゴリ | サービス | 役割 | サブネット |
| :--- | :--- | :--- | :--- |
| **Compute** | **Amazon EC2** | Istio Service Mesh や AIエージェント関連サービスのホスト | プライベート |
| **Networking** | **AWS Load Balancer (ALB)** | 外部からのアクセスエンドポイント | パブリック |
| **Security** | **Amazon Cognito** | ユーザー認証・認可 | パブリック |

<br>
この構成では、n8n 本体や LLM（Ollama）を安全な Private Subnet に隔離しつつ、Public Subnet に配置したロードバランサーを玄関として、Istio Service Mesh を通じてトラフィックを制御しています。ユーザーがブラウザから https://n8n.example.com にアクセスすると、まず Public Subnet に配置されたロードバランサー（図内の紫のアイコン）がリクエストを受信します。インターネットからのトラフィックを直接受け取り、SSL/TLS の終端や、バックエンド（Private Subnet）への転送を行うことになります。<br><br>

次に、ALBをロードバランサーとして配置し、受信したトラフィックを Private Subnet 内で動作する Istio Ingress Gateway Pod へ転送します。Ingress Gateway は、Service Meshの入り口として機能します。図中の n8n-credential（Secret）を参照して、ここで改めて高度な SSL 終端や、パスベースのルーティング判断を行います。また、ロードバランサーと Ingress Gateway を分けることで、AWS インフラ層の制御と、Kubernetes 内部のサービス制御（Istio）を分離することで管理の簡易化を目指すものです。<br>

Ingress Gateway に届いたリクエストは、Istio の設定オブジェクトである VirtualService によって、適切な宛先へ振り分けられます。ここでは、ホスト名（n8n.example.com）に基づいて、宛先となる n8n の Service や Pod を特定します。<br>

最終的に、リクエストが n8n Pod へ到達することでユーザーはエディタや Webhook を利用可能になります。しかし、n8n をプライベートサブネットに配置しても、外部公開用 ALB を経由する限りサービスが実質的にインターネットへ露出してしまう点が大きな課題です。この構成でセキュリティを高めるには Amazon Cognito 等との連携が不可欠ですが、ユーザープールの維持や OIDC 設定といった膨大な運用負荷は避けられません。また、認証を一段増やしても公開状態であることに変わりはなく、エンドポイントへの不正アクセスや未知の脆弱性に対する抜本的な対策にはならないという不安が残ります。<br>

### 2-1. アーキテクチャの改善

そこで、以下のようにWorkSpacesを介したセキュアなデスクトップ接続により、外部公開のリスクを排除し、厳格なガバナンスとコンプライアンスを実現します。WorkSpaces上のセキュアなデスクトップ環境で操作を完結させることで、ローカルPCへのデータ保存を制限し、情報漏洩リスクを最小化することも狙いです。<br><br>

<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-aws2.drawio.png" width="720">

| カテゴリ | サービス | 役割 | サブネット |
| :--- | :--- | :--- | :--- |
| **Compute** | **Amazon EC2** | Istio Service Mesh や AIエージェント関連サービスのホスト | プライベート |
| **Networking** | **AWS Load Balancer (NLB)** | Amazon Workspacesからのアクセスエンドポイント | プライベート |
| **Directory Service** | **Simple AD or AWS Managed Microsoft AD** | Amazon Workspacesのユーザー認証基盤 | プライベート |
| **User Access** | **Amazon WorkSpaces** | AIエージェント環境へアクセスするためのVDI環境 | プライベート |

<br><br>

### 2-2. n8nの公開手法について

オンプレミス環境では MetalLB を用いた外部公開が一般的ですが、本構成では AWS のマネージドサービスを最大限活用し、AWS WorkSpaces + NLB + Istio による強固な閉域アクセス環境を実現します。外部公開 URL をグローバル DNS に登録する必要がなく、VPC 内でのみ有効なプライベートドメイン（例: `n8n.internal`）での運用が可能になります。これにより、n8n のログイン画面や Webhook エンドポイントがインターネット上の攻撃者から完全に不可視（Invisible）となり、インフラレベルでのセキュリティが飛躍的に向上します。

ユーザー認証には Simple AD と連携した WorkSpaces を採用し、物理的に分離されたデスクトップ環境からのアクセスのみを許可。ネットワーク階層では NLB (Network Load Balancer) によるプロトコルパススルーを行い、最終的な HTTPS 終端を Istio Ingress Gateway で実施することで、証明書管理の柔軟性とゼロトラストなトラフィック制御を両立させています。

本アーキテクチャを採用することで、従来のオンプレミス環境や標準的な外部公開構成と比較して、以下の3つの大きなメリットを享受できます。

- ゼロトラストモデルの体現:
  「認証されたデバイスおよびデスクトップ（AWS WorkSpaces）」からのアクセスのみを信頼する構造を構築しています。Simple ADによるユーザー認証を通過したWorkSpaces環境という「信頼できる境界」をゲートウェイにすることで、単なるID/PW認証を超えた、デバイスレベルでの厳格なアクセス制御を実現します。

- 完全閉域化による攻撃パスの排除:
  n8nのエンドポイントをパブリックなインターネットから完全に隔離します。外部IPを介した接続を物理的に排除し、VPC内部のみで有効なプライベートドメイン（例：n8n.internal）で運用することで、ログイン画面やWebhookがインターネット上のスキャナーや攻撃者から一切「見えない」状態を作り出し、インフラレベルでの安全性を担保します。

- インフラ運用の圧倒的な簡素化:
  ロードバランサーに NLB (Network Load Balancer) を採用することで、アーキテクチャがシンプルになります。通常、ALBでセキュアに公開する際に必要となる「前段でのCognito連携」や「ALB側での複雑な証明書・認証設定」を不要とし、暗号化（HTTPS）の終端や認可のロジックを Istio Ingress Gateway 側に集約・完結させることができます。


## 3. WorkSpaces 環境について

n8n 上で動作する AI エージェント（LangChain 連携や LLM による自動操作）を、自身のオフィス環境から分離された Amazon WorkSpaces 上で運用・操作することには、以下の独自のメリットがあります。

- ガバナンスとリスクの局所化 (Blast Radius Control):
  AI エージェントが万が一予期せぬ動作（誤ったデータの削除や、機密情報の外部への意図しない送信など）を行ったとしても、影響範囲を WorkSpaces 内の閉域ネットワークに封じ込めることができます。自分の PC やオフィスネットワークとは論理的に切り離されているため、社内資産への二次被害を防ぐ「サンドボックス」として機能します。

- 機密情報の露出防止 (Screen/Data Isolation):
  AI エージェントのプロンプトや実行結果には、企業の戦略や重要データが含まれることが多々あります。WorkSpaces を使用することで、これらの情報が個人のローカルデバイスにキャッシュされるのを防ぎます。画面転送のみを行う WorkSpaces の特性により、AI との対話ログが手元の PC に残らず、安全なリモート環境に集約されます。

- AI 実行環境の固定化と安定性:
  AI エージェントを操作する際、ローカル PC のネットワーク不安定性や OS のアップデートに左右されず、常に VPC 内部の安定したインフラから指示を出すことができます。オフィス環境のファイアウォール設定を個別に変更することなく、VPC 内で完結した通信経路を利用できるため、AI エージェントとのセッションの継続性が向上します。

### 認証・アクセスフロー

インターネットにエンドポイントを公開せず、**AWS WorkSpaces** という制御されたデスクトップ環境からのアクセスのみを許可する閉域アクセスモデルが本構成の特徴です。

1. **Access Control (VDI Auth):** ユーザーは **Simple AD** で認証された **AWS WorkSpaces** にログインします。このディレクトリサービスによる認証が、システムへの第一の関門となります。

2. **Request (Internal DNS):** WorkSpaces 上のセキュアなブラウザから、内部専用ドメイン（例: `n8n.internal`）を使用してアクセスします。リクエストは VPC 内の **Route 53 Private Hosted Zone** を介して NLB へ解決されます。

3. **Transport (L4 Passthrough):** **NLB (Network Load Balancer)** がリクエストを受信。L4（TCP）レイヤーで動作するため、HTTPS 暗号化パケットを書き換えることなく、そのまま **Istio Ingress Gateway** へパススルー転送します。

4. **Encryption (TLS Termination):** **Istio Ingress Gateway** が、Kubernetes Secret 内に保持された証明書を用いて HTTPS を終端（SSL復号）します。これにより、証明書管理の柔軟性を Kubernetes 側に集約しています。

5. **Verify & Route:** Istio が `AuthorizationPolicy` を適用。アクセス元が WorkSpaces のサブネット CIDR であることを最終検証し、正規のトラフィックのみを n8n Pod へルーティングします。


---



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





