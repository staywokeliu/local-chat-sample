为了构建一个基于 Amazon Bedrock (Claude 3.5 Sonnet) 并通过 WebSocket 进行流式回答的 Chat App，我们将使用以下架构：

* **前端（Flask，支持 WebSocket）**
* **后端 Lambda（Python）处理 WebSocket 通信并转发到 Claude 3.5**
* **使用 Amazon Bedrock SDK 与 Claude 3.5 Sonnet 通信（支持流式响应）**

以下是项目的整体构建方案：

---

## 🧩 项目结构（简化版）

```
chat-app/
├── frontend/  # Flask app
│   ├── app.py
│   ├── static/
│   │   └── chat.js
│   └── templates/
│       └── chat.html
├── backend/   # Lambda function (WebSocket handler)
│   └── websocket_handler.py
├── shared/
│   └── bedrock_utils.py
```

---

## 📦 依赖项

### ✅ Lambda（部署在 API Gateway 的 WebSocket 支持下）

* `boto3`（调用 Bedrock）
* `aws-lambda-powertools`（可选，日志等）
* Python 3.11+

### ✅ Flask 前端

* Flask
* Flask-SocketIO
* gevent

---

## 🔧 步骤一：后端（Lambda + Bedrock Claude 3.5 Streaming）

### 🔹 `backend/websocket_handler.py`

```python
import boto3
import json
import os

bedrock = boto3.client("bedrock-runtime", region_name="ap-northeast-1")

def lambda_handler(event, context):
    connection_id = event['requestContext']['connectionId']
    domain = event['requestContext']['domainName']
    stage = event['requestContext']['stage']

    body = json.loads(event['body'])
    user_input = body.get("message", "")

    # 使用 Claude 3.5 进行流式响应
    response = bedrock.invoke_model_with_response_stream(
        modelId="anthropic.claude-3-sonnet-20240229-v1:0",
        body=json.dumps({
            "prompt": f"\n\nHuman: {user_input}\n\nAssistant:",
            "max_tokens": 1024,
            "temperature": 0.7,
            "stream": True
        }),
        contentType="application/json",
        accept="application/json"
    )

    # 将响应逐步通过 API Gateway WebSocket 发送回去
    gatewayapi = boto3.client("apigatewaymanagementapi",
                              endpoint_url=f"https://{domain}/{stage}")
    
    for chunk in response['body']:
        data = json.loads(chunk['chunk']['bytes'].decode())
        message = data.get("completion", "")
        gatewayapi.post_to_connection(ConnectionId=connection_id,
                                      Data=message)
    
    return {'statusCode': 200}
```

---

## 💻 步骤二：前端（Flask + SocketIO）

### 🔹 `frontend/app.py`

```python
from flask import Flask, render_template
from flask_socketio import SocketIO, emit
import json
import threading
import websocket  # websocket-client

app = Flask(__name__)
socketio = SocketIO(app)

@app.route('/')
def index():
    return render_template("chat.html")

@socketio.on('send_message')
def handle_message(message):
    def stream_response():
        ws = websocket.WebSocket()
        ws.connect("wss://your-api-gateway-url/dev")

        ws.send(json.dumps({"message": message}))
        while True:
            data = ws.recv()
            if not data:
                break
            socketio.emit('receive_message', data)

        ws.close()

    threading.Thread(target=stream_response).start()

if __name__ == '__main__':
    socketio.run(app, debug=True)
```

---

### 🔹 `templates/chat.html`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Claude 3.5 Chat</title>
    <script src="https://cdn.socket.io/socket.io-3.0.0.min.js"></script>
</head>
<body>
    <h2>Claude Chat</h2>
    <div id="chat"></div>
    <input id="input" autocomplete="off" /><button onclick="send()">Send</button>

    <script>
        const socket = io();
        const chat = document.getElementById('chat');

        socket.on('receive_message', function(msg) {
            const div = document.createElement('div');
            div.textContent = msg;
            chat.appendChild(div);
        });

        function send() {
            const input = document.getElementById('input');
            socket.emit('send_message', input.value);
            input.value = '';
        }
    </script>
</body>
</html>
```

---

## 🔐 IAM 权限建议（Lambda 执行角色）

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModelWithResponseStream",
    "execute-api:ManageConnections"
  ],
  "Resource": "*"
}
```

---

## 🚀 部署步骤（概要）

1. **API Gateway 创建 WebSocket API**（设置路由如 `$connect`, `$default`, `$disconnect`）
2. **Lambda 部署为 WebSocket 后端集成**
3. **前端 Flask 使用 WebSocket 客户端连接 API Gateway**
4. **前端通过 Flask SocketIO 接收流式数据展示**

---

如果你希望我生成完整的 deployable zip 文件、Flask 项目打包方式，或者用 CDK/Terraform 管理基础设施，我也可以继续帮你。是否需要这些进一步的内容？
