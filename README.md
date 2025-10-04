# n8nで開発する、RAGを活用したAIエージェントによるSlackチャットボット
# 0. 必要なもの
mac book air m3/m4 (above 24GB memory)
# 1. ゴール
一般的なクラウドサービス（AWS、Azure、GCPなど）のマネージドサービスを使わずに、オンプレミス環境で、Kubernetes上にAIエージェントによるSlackチャットボットを構築します。この際、サービスメッシュを使い、HTTPS環境を構築することでセキュアな実行環境を構築します。<br><br>
The goal is to build a Slack chatbot with an AI agent on Kubernetes in an on-premises environment without using managed services from common cloud services (AWS, Azure, GCP, etc.).In this case, a secure execution environment is created by using a service mesh and building an HTTPS environment.<br>
### 1-1. AIエージェントの完成イメージ
- 左半分：任意のpdfをアップロードし、ベクトルデータベース化するフロー<br>
- 右半分：Slackのチャネルに投稿があった場合、それをトリガーにして投稿内容をLLMが理解し、保存されているナレッジを使用して、ユーザーからの質問にSlackで回答するフロー<br><br>
- Left half: A flow for uploading any PDF and creating a vector database.<br>
- Right half: When a post is made to a Slack channel, it triggers LLM to understand the post and use stored knowledge to answer user questions via Slack.<br>
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-slack-RAG.png" width="960">

### 1-2. Slackチャットボットの受け答えイメージ
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/slack-bot.png" width="960">

# 2. 構築までの流れ(仮想マシン部)
### 2-1. Install Hypervisor
>https://www.oracle.com/jp/virtualization/technologies/vm/downloads/virtualbox-downloads.html

### 2-2. Install Vagrant
>https://developer.hashicorp.com/vagrant/install

### 2-3. Install git & git clone
>https://git-scm.com/downloads
```
git clone https://github.com/developer-onizuka/n8n-istio
cd n8n-istio
```
### 2-4. Roll out Virtual Machine
### 2-4-1. Master node / Worker node
```
cd kubernetes
vagrant up
cd ..
```
### 2-4-2. NFS Server
```
cd nfs
vagrant up
cd ..
```
# 3. 構築までの流れ(Kubernetes部)
### 3-1. Login Master node & git clone
```
cd kubernetes
vagrant ssh master
git clone https://github.com/developer-onizuka/n8n-istio
cd n8n-istio
```
### 3-2. Confirm Kubernetes Cluster
```
kubectl get nodes
```
```
$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   39m   v1.33.5
worker1   Ready    node            38m   v1.33.5
```
### 3-3. Setup LoadBalancer
ロードバランサーに割り当てる IP アドレスの範囲を指定します。<br><br>
Specify the range of IP addresses to be assigned to the load balancer.<br>
```
kubectl apply -f metallb-ipaddress.yaml
```
### 3-4. Install CSI driver for NFS
```
./install-csi-driver.sh
```
### 3-5. Roll out StorageClass
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
### 3-6. Roll out PV
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
# 4. 構築までの流れ(Network部)
### 4-1. Install istio
Istioは、Kubernetes環境で稼働するサービスメッシュを実現するためのオープンソースプラットフォームです。マイクロサービスの接続、セキュリティ、管理、監視を担い、トラフィックルーティング（A/Bテストなど）、認証認可、メトリクス収集といった高度な機能を提供し、開発者ではなくインフラ側でサービス間通信の複雑さを扱えるようにします。今回は、HTTPS環境を構築する際に、IstioのIngress Gatewayを活用します。Ingress Gatewayは、Kubernetesクラスターの外部からのトラフィックを受け付ける Istioの入り口であり、特にセキュリティとトラフィック管理において重要な役割を果たします。<br><br>
Istio is an open-source platform that enables a service mesh running in a Kubernetes environment. It handles the connection, security, management, and monitoring of microservices, providing advanced capabilities like traffic routing (e.g., A/B testing), authentication, authorization, and metrics collection. This approach allows the infrastructure layer, rather than the developers, to manage the complexity of inter-service communication.For this project, we will leverage Istio's Ingress Gateway when setting up the HTTPS environment. The Ingress Gateway is the entry point for Istio, receiving traffic from outside the Kubernetes cluster, and plays a crucial role, particularly in security and traffic management.<br>
```
mkdir work
cd work/
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.27.1/
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
### 4-2. Roll out Ingress Gateway
```
kubectl apply -f istio-n8n.yaml 
```
```
$ kubectl get virtualservices.networking.istio.io 
NAME                 GATEWAYS          HOSTS                 AGE
n8n-virtualservice   ["n8n-gateway"]   ["n8n.example.com"]   92m
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.101.123.63   192.168.33.3   15021:32287/TCP,80:31347/TCP,443:30994/TCP   46h
```
### 4-3. Create secret for HTTPS
ここでは、Kubernetes環境、特にIstioサービスメッシュ内で安全な HTTPS通信を確立するための 自己署名証明書を作成し、KubernetesのSecretとして保存する一連のプロセスを実行しています。<br><br>
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

### 4-4. Create tunnel with ngrok
ngrokは、Webhookの重要な課題を解決するため、ローカルなn8n環境に不可欠なツールです。ローカルなn8n環境では、n8nインスタンスはルーターとファイアウォールの背後に隠れてしまい、Slackなどの外部サービスからは直接アクセスできません。ngrokは、パブリックインターネットアドレスからローカルマシンへの安全なトンネルを作成することでこの問題を解決します。これにより、外部サービスがローカルで実行されているn8nのWebhook URLにHTTPリクエストを送信し、n8nインスタンスへのWebhookを可能にします。<br><br>
Ngrok is an essential tool for local n8n development because it solves a critical Webhook challenge. When developing locally, your n8n instance is hidden behind your router and firewall, making it inaccessible to external services. ngrok addresses this by creating a secure tunnel from a public internet address to your local machine, allowing external services (like Slack or payment processors) to successfully send HTTP requests to your locally running n8n Webhook URL for seamless testing and debugging.<br>
```
kubectl apply -f ngrok-secret.yaml
kubectl apply -f ngrok.yaml 
```
```
$ kubectl get secrets 
NAME              TYPE     DATA   AGE
ngrok-authtoken   Opaque   1      20h
```
# 5. 構築までの流れ(コンテナ部)
### 5-1. Roll out Ollama & n8n
n8n-ingress.yaml ファイルで、WEBHOOK_TUNNEL_URL の値を、次のコマンドを実行した結果に設定 (または更新) します。<br><br>
In the n8n-ingress.yaml file, set (or update) the WEBHOOK_TUNNEL_URL value to the result of running the following command:<br>
```
$ kubectl logs ngrok-tunnel-client-74697dd844-8hzc8 | jq -r 'select(.url != null) | .url'
https://xxxxx.ngrok-free.dev
```
After edit the file of n8n-ingress.yaml, let's apply it in the kubernetes:
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
### 5-2. Confirm Services
```
$ kubectl get services -o wide
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE   SELECTOR
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP     10d   <none>
svc-mongodb   ClusterIP   10.111.235.84    <none>        27017/TCP   10d   app=mongodb
svc-n8n       ClusterIP   10.109.150.138   <none>        5678/TCP    87m   app=n8n
svc-ngrok     ClusterIP   10.108.101.133   <none>        4040/TCP    90m   app=ngrok
svc-ollama    ClusterIP   10.100.121.176   <none>        11434/TCP   10d   app=ollama
```





