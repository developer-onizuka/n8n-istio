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
この構成では、n8n 本体や LLM（Ollama）を安全な Private Subnet に隔離しつつ、Public Subnet に配置したロードバランサーを入り口として、Istio Service Mesh を通じてトラフィックを制御しています。ユーザーがブラウザから https://n8n.example.com にアクセスすると、まず Public Subnet に配置されたロードバランサー（図内の紫のアイコン）がリクエストを受信します。インターネットからのトラフィックを直接受け取り、SSL/TLS の終端や、バックエンド（Private Subnet）への転送を行うことになります。<br><br>

次に、ALBをロードバランサーとして配置し、受信したトラフィックを Private Subnet 内で動作する Istio Ingress Gateway Pod へ転送します。Ingress Gateway は、Service Meshの入り口として機能します。図中の n8n-credential（Secret）を参照して、ここで改めて高度な SSL 終端や、パスベースのルーティング判断を行います。また、ロードバランサーと Ingress Gateway を分けることで、AWS インフラ層の制御と、Kubernetes 内部のサービス制御（Istio）を分離することで管理の簡易化を目指すものです。<br>

Ingress Gateway に届いたリクエストは、Istio の設定オブジェクトである VirtualService によって、適切な宛先へ振り分けられます。ここでは、ホスト名（n8n.example.com）に基づいて、宛先となる n8n の Service や Pod を特定します。<br>

最終的に、リクエストが n8n Pod へ到達することでユーザーはエディタや Webhook を利用可能になります。しかし、n8n をプライベートサブネットに配置しても、外部公開用 ALB を経由する限りサービスが実質的にインターネットへ露出してしまう点が大きな課題です。この構成でセキュリティを高めるには Amazon Cognito 等との連携が不可欠ですが、ユーザープールの維持や OIDC 設定といった膨大な運用負荷は避けられません。また、認証を一段増やしても公開状態であることに変わりはなく、エンドポイントへの不正アクセスや未知の脆弱性に対する抜本的な対策にはならないという不安が残ります。<br>

そこで、以下のようにWorkSpacesを介したセキュアなデスクトップ接続により、外部公開のリスクを排除し、厳格なガバナンスとコンプライアンスを実現します。WorkSpaces上のセキュアなデスクトップ環境で操作を完結させることで、ローカルPCへのデータ保存を制限し、情報漏洩リスクを最小化することも狙いです。<br><br>

<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-aws2.drawio.png" width="720">

| カテゴリ | サービス | 役割 | サブネット |
| :--- | :--- | :--- | :--- |
| **Compute** | **Amazon EC2** | Istio Service Mesh や AIエージェント関連サービスのホスト | プライベート |
| **Networking** | **AWS Load Balancer (NLB)** | Amazon Workspacesからのアクセスエンドポイント | プライベート |
| **Directory Service** | **Simple AD or AWS Managed Microsoft AD** | Amazon Workspacesのユーザー認証基盤 | プライベート |
| **User Access** | **Amazon WorkSpaces** | AIエージェント環境へアクセスするためのVDI環境 | プライベート |

<br>

なお、オンプレミス環境では MetalLB を用いた外部公開が一般的ですが、本構成では AWS のマネージドサービスを最大限活用し、AWS WorkSpaces + NLB + Istio による強固な閉域アクセス環境を実現します。外部公開 URL をグローバル DNS に登録する必要がなく、VPC 内でのみ有効なプライベートドメイン（例: `n8n.internal`）での運用が可能になります。これにより、n8n のログイン画面や Webhook エンドポイントがインターネット上の攻撃者から見えなくなり、インフラレベルでのセキュリティが向上します。

ユーザー認証には Simple AD と連携した WorkSpaces を採用し、物理的に分離されたデスクトップ環境からのアクセスのみを許可し、ネットワーク階層では NLB (Network Load Balancer) によるプロトコルパススルーを行い、最終的な HTTPS 終端を Istio Ingress Gateway で実施することで、証明書を自分で管理することができ、特定のクラウドプロバイダーの仕様や制約に縛られることなく、要件に応じた最適な認証局を自由に選択できるようになります。

## 3. Amazon WorkSpaces について

n8n 上で動作する AI エージェント（LangChain 連携や LLM による自動操作）を、自身のオフィス環境から分離された Amazon WorkSpaces 上で運用・操作することには、以下の独自のメリットがあります。

- 機密情報の露出防止 (Screen/Data Isolation):<br>
  AI エージェントのプロンプトや実行結果には、企業の戦略や重要データが含まれることが多々あります。WorkSpaces を使用することで、これらの情報が個人のローカルデバイスにキャッシュされるのを防ぎます。画面転送のみを行う WorkSpaces の特性により、AI との対話ログが手元の PC に残らず、安全なリモート環境に集約されます。

- AI 実行環境の固定化と安定性:<br>
  AI エージェントを操作する際、ローカル PC のネットワーク不安定性や OS のアップデートに左右されず、常に VPC 内部の安定したインフラから指示を出すことができます。オフィス環境のファイアウォール設定を個別に変更することなく、VPC 内で完結した通信経路を利用できるため、AI エージェントとのセッションの継続性が向上します。

### 3-1. 認証・アクセスフロー

インターネットにエンドポイントを公開せず、**AWS WorkSpaces** という制御されたデスクトップ環境からのアクセスのみを許可する閉域アクセスモデルが本構成の特徴です。

1. **Access Control (VDI Auth):** <br>
ユーザーは **Simple AD** で認証された **AWS WorkSpaces** にログインします。このディレクトリサービスによる認証が、システムへの第一の関門となります。

2. **Request (Internal DNS):** <br>
WorkSpaces 上のセキュアなブラウザから、内部専用ドメイン（例: `n8n.internal`）を使用してアクセスします。リクエストは VPC 内の **Route 53 Private Hosted Zone** を介して NLB へ解決されます。

4. **Transport (L4 Passthrough):** <br>
**NLB (Network Load Balancer)** がリクエストを受信。L4（TCP）レイヤーで動作するため、HTTPS 暗号化パケットを書き換えることなく、そのまま **Istio Ingress Gateway** へパススルー転送します。

6. **Encryption (TLS Termination):** <br>
**Istio Ingress Gateway** が、Kubernetes Secret 内に保持された証明書を用いて HTTPS を終端（SSL復号）します。これにより、証明書管理の柔軟性を Kubernetes 側に集約しています。

7. **Verify & Route:** <br>
Istio が `AuthorizationPolicy` を適用。アクセス元が WorkSpaces のサブネット CIDR であることを最終検証し、正規のトラフィックのみを n8n Pod へルーティングします。


# 4. 構築までの流れ (Kubernetes部)
### 4-1. Login Master node & git clone
```
cd kubernetes
vagrant ssh master
git clone https://github.com/developer-onizuka/n8n-istio
cd n8n-istio
```
### 4-2. Confirm Kubernetes Cluster
```
kubectl get nodes
```
```
$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   39m   v1.33.5
worker1   Ready    node            38m   v1.33.5
```
### 4-3. Install CSI driver for NFS
```
./install-csi-driver.sh
```
### 4-4. Roll out StorageClass
```
kubectl apply -f storageclass-vm-nfs.yaml
kubectl apply -f storageclass-vm-nfs-n8n.yaml
```
```
$ kubectl get sc
NAME                       PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-vm-csi (default)       nfs.csi.k8s.io   Delete          Immediate           false                  8s
nfs-vm-csi-n8n (default)   nfs.csi.k8s.io   Delete          Immediate           false                  6s
```
### 4-5. Roll out PV
```
kubectl apply -f pvc-nfs-ollama.yaml
kubectl apply -f pvc-nfs-n8n.yaml
```
```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS     VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-6614e902-7abc-4865-acc9-f2420f338daa   5Gi        RWX            Delete           Bound    default/pvc-nfs-n8n      nfs-vm-csi-n8n   <unset>                          66s
pvc-8b0667ba-1848-40e5-89f5-0ecaaa0c901d   20Gi       RWX            Delete           Bound    default/pvc-nfs-ollama   nfs-vm-csi       <unset>                          4s
```
### 4-6. NVIDIA GPU Operator
See following:<br>
- https://github.com/developer-onizuka/NVIDIA-GPU-Operator

# 5. 構築までの流れ　(Service Mesh部)
### 5-1. Install istio
Istioは、Kubernetes環境で稼働するService Meshを実現するためのオープンソースプラットフォームです。マイクロサービスの接続、セキュリティ、管理、監視を担い、トラフィックルーティング（A/Bテストなど）、認証認可、メトリクス収集といった高度な機能を提供し、開発者ではなくインフラ側でサービス間通信の複雑さを扱えるようにします。今回は、HTTPS環境を構築する際に、IstioのIngress Gatewayを活用します。Ingress Gatewayは、Kubernetesクラスターの外部からのトラフィックを受け付ける Istioの入り口であり、特にセキュリティとトラフィック管理において重要な役割を果たします。<br><br>
Istio is an open-source platform that enables a service mesh running in a Kubernetes environment. It handles the connection, security, management, and monitoring of microservices, providing advanced capabilities like traffic routing (e.g., A/B testing), authentication, authorization, and metrics collection. This approach allows the infrastructure layer, rather than the developers, to manage the complexity of inter-service communication.For this project, we will leverage Istio's Ingress Gateway when setting up the HTTPS environment. The Ingress Gateway is the entry point for Istio, receiving traffic from outside the Kubernetes cluster, and plays a crucial role, particularly in security and traffic management.<br>
```
mkdir work
cd work/
curl -L https://istio.io/downloadIstio | sh -
cd $(ls)
export PATH=$PWD/bin:$PATH
istioctl version
istioctl install --set profile=default -y
kubectl label namespace default istio-injection=enabled --overwrite 
```
```
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS      AGE
istio-ingressgateway-69579785ff-w6s8x   1/1     Running   1 (20h ago)   46h
istiod-7d4f74889d-xvjr4                 1/1     Running   1 (20h ago)   46h
```
### 5-2. Roll out Ingress Gateway
```
kubectl apply -f istio-n8n.yaml 
```
```
$ kubectl get virtualservices.networking.istio.io 
NAME                 GATEWAYS          HOSTS                 AGE
n8n-virtualservice   ["n8n-gateway"]   ["n8n.example.com"]   92m
$ kubectl get svc -n istio-system 
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.100.57.253    <pending>     15021:31316/TCP,80:30636/TCP,443:30103/TCP   89s
istiod                 ClusterIP      10.103.135.225   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        103s
```
### 5-3. Create secret for HTTPS
ここでは、Kubernetes環境、特にIstioのService Mesh内で安全なHTTPS通信を確立するための自己署名証明書を作成し、KubernetesのSecretとして保存する一連のプロセスを実行しています。<br><br>
This sequence of commands performs the process of creating a self-signed TLS/SSL certificate and storing it as a Kubernetes Secret to enable secure HTTPS communication within a Kubernetes environment utilizing Istio.<br>
```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
openssl req -out n8n.example.com.csr -newkey rsa:2048 -nodes -keyout n8n.example.com.key -subj "/CN=n8n.example.com/O=n8n organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in n8n.example.com.csr -out n8n.example.com.crt
```
```
kubectl create -n istio-system secret tls n8n-credential --key=n8n.example.com.key --cert=n8n.example.com.crt
```
```
$ kubectl get secrets -n istio-system 
NAME              TYPE                DATA   AGE
istio-ca-secret   istio.io/ca-root    5      46h
n8n-credential    kubernetes.io/tls   2      42h
```

### 5-4. Create tunnel with ngrok
ngrokは、Webhookの重要な課題を解決するため、ローカルなn8n環境に不可欠なツールです。ローカルなn8n環境では、n8nインスタンスはルーターとファイアウォールの背後に隠れてしまい、Slackなどの外部サービスからは直接アクセスできません。ngrokは、パブリックインターネットアドレスからローカルマシンへの安全なトンネルを作成することでこの問題を解決します。これにより、外部サービスがローカルで実行されているn8nのWebhook URLにHTTPリクエストを送信し、n8nインスタンスへのWebhookを可能にします。<br><br>
Ngrok is an essential tool for local n8n development because it solves a critical Webhook challenge. When developing locally, your n8n instance is hidden behind your router and firewall, making it inaccessible to external services. ngrok addresses this by creating a secure tunnel from a public internet address to your local machine, allowing external services (like Slack or payment processors) to successfully send HTTP requests to your locally running n8n Webhook URL for seamless testing and debugging.<br><br>

https://ngrok.com/ でライセンスを取得し、ngrok-secret.yamlにあなたのトークンを登録して、下記のコマンドを実行してください。<br>
```
kubectl apply -f ngrok-secret.yaml
kubectl apply -f ngrok.yaml 
```
```
$ kubectl get secrets 
NAME              TYPE     DATA   AGE
ngrok-authtoken   Opaque   1      20h
```
# 6. 構築までの流れ (コンテナ部)
### 6-1. Roll out Ollama & n8n
n8n-ingress.yaml ファイルで、WEBHOOK_TUNNEL_URLの値を、次のコマンドを実行した結果に設定 (または更新) します。<br><br>
In the n8n-ingress.yaml file, set (or update) the WEBHOOK_TUNNEL_URL value to the result of running the following command:<br>
```
$ kubectl logs ngrok-tunnel-client-74697dd844-8hzc8 | jq -r 'select(.url != null) | .url'
https://xxxxxxxxxx.ngrok-free.dev
```
After edit the file of n8n-ingress.yaml, let's apply it in the kubernetes:<br>
If you are using GPU, then apply "ollama-gpu.yaml" instead of ollama.yaml.
```
kubectl apply -f ollama.yaml
kubectl apply -f n8n-ingress.yaml
```
```
$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS      AGE   IP              NODE      NOMINATED NODE   READINESS GATES
n8n-bcc459b78-wg77l                    2/2     Running   0             94m   10.10.235.139   worker1   <none>           <none>
ngrok-tunnel-client-74697dd844-7ms4n   2/2     Running   0             98m   10.10.235.137   worker1   <none>           <none>
ollama-6c988c64c6-qgsdm                1/1     Running   7 (20h ago)   10d   10.10.235.173   worker1   <none>           <none>
```
### 6-2. Confirm Services
```
$ kubectl get services -o wide
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE   SELECTOR
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP     10d   <none>
svc-mongodb   ClusterIP   10.111.235.84    <none>        27017/TCP   10d   app=mongodb
svc-n8n       ClusterIP   10.109.150.138   <none>        5678/TCP    87m   app=n8n
svc-ngrok     ClusterIP   10.108.101.133   <none>        4040/TCP    90m   app=ngrok
svc-ollama    ClusterIP   10.100.121.176   <none>        11434/TCP   10d   app=ollama
```
### 6-3. Download the llama in Ollama as a brain
```
kubectl exec -it <your-ollama-pod-name> -- ollama pull llama3.2:3b
```
```
$ kubectl exec -it pods/ollama-6c988c64c6-7fxfp -- ollama pull llama3.2:3b
pulling manifest 
pulling dde5aa3fc5ff: 100% ▕██████████████████████████████████████▏ 2.0 GB                         
pulling 966de95ca8a6: 100% ▕██████████████████████████████████████▏ 1.4 KB                         
pulling fcc5a6bec9da: 100% ▕██████████████████████████████████████▏ 7.7 KB                         
pulling a70ff7e570d9: 100% ▕██████████████████████████████████████▏ 6.0 KB                         
pulling 56bb8bd477a5: 100% ▕██████████████████████████████████████▏   96 B                         
pulling 34bb5ab01051: 100% ▕██████████████████████████████████████▏  561 B                         
verifying sha256 digest 
writing manifest 
success
```
You can confirm if the model can work well:
```
$ kubectl exec -it pods/ollama-87cbf6d6c-tc9x7 -- ollama run llama3.2:3b
>>> Send a message (/? for help)
```
So, you can find the following Memory Usage while running ollama:
```
+-----------------------------------------------------------------------------------------+
Mon Oct 13 08:55:30 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.65.06              Driver Version: 580.65.06      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Quadro P1000                   On  |   00000000:03:00.0 Off |                  N/A |
| 58%   71C    P0            N/A  /  N/A  |    2705MiB /   4096MiB |     95%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

### 6-3-1. (Optional) Download gpt-oss:20b
```
kubectl exec -it <your-ollama-pod-name> -- ollama pull gpt-oss:20b
```
```
$ kubectl exec -it pods/ollama-87cbf6d6c-hzz9b -- ollama pull gpt-oss:20b
pulling manifest 
pulling e7b273f96360: 100% ▕██████████████████████████████████████▏  13 GB                         
pulling fa6710a93d78: 100% ▕██████████████████████████████████████▏ 7.2 KB                         
pulling f60356777647: 100% ▕██████████████████████████████████████▏  11 KB                         
pulling d8ba2f9a17b3: 100% ▕██████████████████████████████████████▏   18 B                         
pulling 776beb3adb23: 100% ▕██████████████████████████████████████▏  489 B                         
verifying sha256 digest 
writing manifest 
success 
```
### 6-3-2. (Optional) Download llama3.1:8b-instruct-q6_K
llama3.1:8b-instruct-q6_Kは、Meta Llama 3.1の80億パラメータの対話（Instruct）モデルを、6ビットKuantization（q6_K）で圧縮したバージョンです。高性能を維持しつつ省メモリで、ローカル環境で気軽に試用・実行したいユーザーに推奨されるモデルです。なお、llama3.1:8b-instruct-fp16は、Meta社がリリースした量子化されていないモデルで、このFP16モデルをベースにして、重みを6ビット（Q6_K）という低い精度に量子化したものが、llama3.1:8b-instruct-q6_Kになります。<br><br>
llama3.1:8b-instruct-q6_K is a version of the 8 billion parameter interaction (Instruct) model from Meta Llama 3.1, compressed using 6-bit quantization (q6_K). This model is recommended for users who want to easily try it out in a local environment while maintaining high performance and using less memory.
- https://ollama.com/library/llama3.1/tags
```
kubectl exec -it <your-ollama-pod-name> -- ollama pull llama3.1:8b-instruct-q6_K
```
```
$ kubectl exec -it ollama-87cbf6d6c-b6sg2 -- ollama pull llama3.1:8b-instruct-q6_K
pulling manifest 
pulling 2eda6fab632c: 100% ▕██████████████████████████████████████▏ 6.6 GB                         
pulling 948af2743fc7: 100% ▕██████████████████████████████████████▏ 1.5 KB                         
pulling 0ba8f0e314b4: 100% ▕██████████████████████████████████████▏  12 KB                         
pulling 56bb8bd477a5: 100% ▕██████████████████████████████████████▏   96 B                         
pulling ad688d3b0503: 100% ▕██████████████████████████████████████▏  485 B                         
verifying sha256 digest 
writing manifest 
success
```
### 6-4. Download the embed model in Ollama for RAG
```
kubectl exec -it <your-ollama-pod-name> -- ollama pull nomic-embed-text:latest 
```
```
$ kubectl exec -it pods/ollama-6c988c64c6-qgsdm -- ollama pull nomic-embed-text:latest
pulling manifest 
pulling 970aa74c0a90: 100% ▕██████████████████████████████████████▏ 274 MB                         
pulling c71d239df917: 100% ▕██████████████████████████████████████▏  11 KB                         
pulling ce4a164fc046: 100% ▕██████████████████████████████████████▏   17 B                         
pulling 31df23ea7daa: 100% ▕██████████████████████████████████████▏  420 B                         
verifying sha256 digest 
writing manifest 
success 
```
```
$ kubectl exec -it pods/ollama-87cbf6d6c-hzz9b -- ollama run llama3.1:8b-instruct-q6_K
>>> Send a message (/? for help)
```
```
Sun Nov  9 14:21:42 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.95.05              Driver Version: 580.95.05      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Quadro P4000                   On  |   00000000:03:00.0 Off |                  N/A |
| 50%   44C    P8              5W /  105W |    6809MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```
# 7. AI Agent for Slack Chatbot
### 7-1. RAG
まずは、以下のURLを参考に、RAG部分(左半分)のフローを作成してください。<br>
- https://github.com/developer-onizuka/n8n-ollama?tab=readme-ov-file#9-ai-agent-for-rag
### 7-2. Slack Trigger
### 7-2-1. Slack Bot User OAuth Tokenの取得
以下のURLで、右上の[Create New App]をクリックし、[From scratch]を選択します。<br>
- https://api.slack.com/apps/
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/slack-API.png" width="480">

ここでは、アプリの名前とアプリを動作させるワークスペースを選択します。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/create-App.png" width="480">

アプリケーションの設定画面にて、OAuth＆Permissionsを選択します。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/OAuth-Permissions.png" width="480">

Slackアプリに付与する、権限をスコープとして設定します。ここで与えた権限は、Slackワークスペースにおけるアプリの動作を制限するためのものです。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Bot-Toke-Scopes.png" width="480">

ここでは、ワークスペースにインストールすることになるアプリに関するOAuthトークンを作成します。[Install to xxx]をクリックします。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/OAuth-Tokens.png" width="480">

許可のボタンをクリックします。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/App-Workspace.png" width="480">

Bot User OAuth Tokenをコピーします。後々、このトークンを使用してアプリを認証することになります。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Bot-User-OAuth-Token.png" width="480">

### 7-2-2. Slack Bot User OAuth Tokenの取得
以下のURLで、Slackを起動します。<br>
- https://slack.com/intl/ja-jp/

以下のようにSlack Channelで、アプリを追加します。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Adding-App-in-the-channel.png" width="480">
<br>↓↓↓↓↓<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Adding-App-in-the-channel2.png" width="480">

### 7-2-3. 認証設定
Slack TriggerノードをCanvas上に置きます。Slackの設定画面にて、ChannelのIDを入力します。ChannelのIDはSlackが立ち上がっているWebのURL "https://app.slack.com/client/" のCから始まる最後の12桁になります。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Slack-Trigger.png" width="960">

次に、[Create new credential]をクリックし、先ほど取得したBot User OAuth Tokenを貼り付けます。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Access-Token.png" width="960">

### 7-3. Edit Field
次の画面を参考に設定をしてください。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Edit-Field.png" width="960">

### 7-4. AI Agent
これはベクターストアを使用するAI エージェントであり、RAG を搭載したチャットボットとして機能します。<br><br>
This is the AI Agent which uses vector store and it works as a chatbot powered by RAG.<br>
<img src="https://github.com/developer-onizuka/n8n-ollama/blob/main/rag-simple-vector-store.png" width="320">

Description:
```
保存されているナレッジを使用して、ユーザーからの質問に回答してください。
```
### 7-5. Slack Send a message
最後にワークフローの最後にAI Agentが作成した回答をSlack Channelに回答するノードを作成します。ここでは、主にChannel IDを設定します。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Send-Message.png" width="960">

Client ID、Secretを設定します。https://api.slack.com/apps/ でアプリを選択し、Basic Informationでこれらの値を確認できます。<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Redirect-URL.png" width="960"><br><br>
表示されているRedirect URL "https://n8n.example.com/rest/oauth2-credential/callback" を、Slackアプリの「OAuth & Permissions」に追加してください。リダイレクトURLは、OAuth認証フローの承認結果を、安全かつ正確にクライアントアプリケーションであるn8nノード（Slack Send a message）に伝えるために用いられます。具体的には、Slackの権限を操作できるユーザーがブラウザ上で「許可」を押した後、SlackはこのURLに認証コードを付けてリダイレクト（ウェブサイトが自動的に別のページへ移動）します。Istio Ingress GatewayによりHTTPSで暗号化されているため、この重要な認証コードを安全にn8nへ渡し、Slack Send a messageのノードが最終的なアクセストークンを取得できるようになります。なお、このRedirect URLは、外部に公開する必要はありません。（今回の例も、内部環境のみで有効なURLとなっており、Slackの権限を操作できるユーザーがブラウザで到達できるネットワークであればよい。）<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Redirect-URL2.png" width="480">

### 7-6. Event Subscription
https://api.slack.com/apps/ でアプリを選択し、Event Subscription画面に遷移します。
[Enable Events]をOnにします。[Add Bot User Event]をクリックしてapp_mentionを選択します。Requested URLにProduction URLをペーストします。**[Save Changes]をクリックして変更を保存します(重要)。**
**また、ワークフローがActiveな状態でない場合は、このProduction URLはVerifiedとなりません(重要)。** <br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/Event-Subscription.png" width="480">

Production URLは、Slack Triggerをダブルクリックして現れる画面からコピーします。**ここではlocalhostから始まるURLですが、localhostの部分は、n8n-ingress.yaml内で定義したWEBHOOK_TUNNEL_URLと置き換えます(重要)。**<br><br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/POST-URL.png" width="480">





