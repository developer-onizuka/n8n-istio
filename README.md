# n8n + istio + AWS Cognito + AWS WorkSpaces Web

## 1. アーキテクチャ概要

本構成では、n8n本体をプライベートサブネットに完全に隠蔽し、外部からのアクセスを **Amazon Cognito** で認証。さらに、管理者によるメンテナンスを **Amazon WorkSpaces Web** 経由に限定することで、最高水準のセキュリティを担保します。

### コンポーネント構成
| カテゴリ | サービス | 役割 |
| :--- | :--- | :--- |
| **Compute** | **Amazon EKS (v1.28+)** | n8n および Istio Service Mesh のホスト |
| **Networking** | **AWS Load Balancer (ALB)** | 外部入口。SSL終端とCognito認証のハンドシェイク |
| **Service Mesh** | **Istio** | Pod間通信の暗号化 (mTLS) と高度なルーティング |
| **Identity** | **Amazon Cognito** | 外部ユーザー・Webhookの認証・認可基盤 |
| **Admin Access** | **Amazon WorkSpaces Web** | 管理者専用のセキュアなブラウザ環境（VPC内接続） |

---

## 2. 外部アクセスの方法論 (Identity-Aware Proxy)

オンプレミス環境では、MetalLBを使うことが多いのですが、今回はAWSの **ALB + Cognito + Istio** による多層防御へ進化させています。

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





