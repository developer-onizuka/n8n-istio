# n8nで開発する、RAGを活用したAIエージェントによるSlackチャットボット

# 1. 必要なもの
mac book air m3/m4 (above 24GB memory)

# 2. ゴール
一般的なクラウドサービス（AWS、Azure、GCPなど）のマネージドサービスを使わずに、オンプレミス環境で、Kubernetes上にAIエージェントによるSlackチャットボットを構築することです。

### AIエージェントのイメージ
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/n8n-slack-RAG.png" width="960">

### Slackチャットボットの受け答えイメージ
<img src="https://github.com/developer-onizuka/n8n-istio/blob/main/slack-bot.png" width="960">


```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
openssl req -out n8n.example.com.csr -newkey rsa:2048 -nodes -keyout n8n.example.com.key -subj "/CN=n8n.example.com/O=n8n organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in n8n.example.com.csr -out n8n.example.com.crt

kubectl create -n istio-system secret tls n8n-credential --key=n8n.example.com.key --cert=n8n.example.com.crt
```

```
$ kubectl logs ngrok-tunnel-client-74697dd844-8hzc8 | jq -r 'select(.url != null) | .url'
https://xxxxx.ngrok-free.dev
```
