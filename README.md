# NginxExample

An example of how in the js_filter handler function s.send(data, flags) works synchronously, but doesn't work asynchronously.

## Prerequisites
(examples given for MacOS)

- Install docker
```shell
brew install docker
```

- Install docker-compose
```shell
brew install docker-compose
```

- Install mosquitto
```shell
brew install mosquitto
```

## Running the example

### Open a console in the NginxExample directory:

- Make sure that you can run the scripts:
```shell
chmod +x ./client/*.sh
```

- Start the app (with rebuilding):
```shell
docker-compose up --build
```
It will show the logs of mosquitto and nginx.

### The case that works 1
#### In console 2:
- Try to subscribe to topic1 (it should work):
```shell
% ./client/sub.sh 1883 client1 topic1
Client client1 sending CONNECT
Client client1 received CONNACK (0)
Client client1 sending SUBSCRIBE (Mid: 1, Topic: topic1, QoS: 0, Options: 0x00)
Client client1 received SUBACK
Subscribed (mid: 1): 0
```
#### In console 3:
- Try to publish to topic1 (with a different client ID!) (it should work):
```shell
% ./client/pub.sh 1883 client2 topic1 qwe
Client client2 sending CONNECT
Client client2 received CONNACK (0)
Client client2 sending PUBLISH (d0, q0, r0, m1, 'topic1', ... (3 bytes))
Client client2 sending DISCONNECT
```
#### In console 2:
- You can check that the subscriber received the message:
```shell
Client client1 received PUBLISH (d0, q0, r0, m0, 'topic1', ... (3 bytes))
topic1 qwe
```
- Press CTRL-C in console 2 to stop the client

### The case that works 2
#### In console 2:
- Try to subscribe to topic1 (it should work):
```shell
% ./client/sub.sh 1883 client1 topic2
Client client1 sending CONNECT
Client client1 received CONNACK (0)
Client client1 sending SUBSCRIBE (Mid: 1, Topic: topic2, QoS: 0, Options: 0x00)
Client client1 received SUBACK
Subscribed (mid: 1): 0
```
#### In console 3:
- Try to publish to topic1 (with a different client ID!) (it should work):
```shell
% ./client/pub.sh 1883 client2 topic2 qwe
Client client2 sending CONNECT
Client client2 received CONNACK (0)
Client client2 sending PUBLISH (d0, q0, r0, m1, 'topic2', ... (3 bytes))
Client client2 sending DISCONNECT
```
#### In console 2:
- You can check that the subscriber received the message:
```shell
Client client1 received PUBLISH (d0, q0, r0, m0, 'topic2', ... (3 bytes))
topic1 qwe
```
- Press CTRL-C in console 2 to stop the client

### The case that doesn't work
#### In console 2:
- Try to subscribe to topic3 (it doesn't work):
```shell
% ./client/sub.sh 1883 client1 topic3
Client client1 sending CONNECT
Client client1 received CONNACK (0)
Client client1 sending SUBSCRIBE (Mid: 1, Topic: topic3, QoS: 0, Options: 0x00)
Client client1 received SUBACK
Subscribed (mid: 1): 0
```
#### In console 3:
- Try to publish to topic1 (with a different client ID!) (it should work):
```shell
% ./client/pub.sh 1883 client2 topic3 qwe
Client client2 sending CONNECT
Client client2 received CONNACK (0)
Client client2 sending PUBLISH (d0, q0, r0, m1, 'topic2', ... (3 bytes))
Client client2 sending DISCONNECT
```
#### In console 2:
- You should see the received "qwe" message but there's nothing.

#### In console 1:
- check the logs:
- It says, "pending events while proxying connection, client: 172.18.0.1, server: 0.0.0.0:1883, upstream: "172.18.0.2:18831", bytes from/to client:36/4, bytes from/to upstream:4/0"
```shell
2023-01-18 14:16:39 mosquitto  | 1674047799: New connection from 172.18.0.3:44676 on port 18831.
2023-01-18 14:16:39 rp         | 2023/01/18 13:16:39 [info] 43#43: *44 client 172.18.0.1:37516 connected to 0.0.0.0:1883
2023-01-18 14:16:39 rp         | 2023/01/18 13:16:39 [info] 43#43: *44 proxy 172.18.0.3:44676 connected to 172.18.0.2:18831
2023-01-18 14:16:39 mosquitto  | 1674047799: New client connected from 172.18.0.3:44676 as client2 (p2, c1, k60).
2023-01-18 14:16:39 mosquitto  | 1674047799: Client client2 closed its connection.
2023-01-18 14:16:39 rp         | 2023/01/18 13:16:39 [info] 43#43: *44 js: calling http://google.com/
2023-01-18 14:16:39 rp         | 172.18.0.1 [18/Jan/2023:13:16:39 +0000] TCP 200 36 4 172.18.0.2:18831
2023-01-18 14:16:39 rp         | 2023/01/18 13:16:39 [info] 43#43: *44 client disconnected, bytes from/to client:36/4, bytes from/to upstream:4/21
2023-01-18 14:16:39 rp         | 2023/01/18 13:16:39 [error] 43#43: *44 pending events while proxying connection, client: 172.18.0.1, server: 0.0.0.0:1883, upstream: "172.18.0.2:18831", bytes from/to client:36/4, bytes from/to upstream:4/0
```

### Explanation

- I have set up the js_filter handler script to listen to the upload messages.
- In the s.on("upload", ...) callback, I check the MQTT packet type and topics.
- If the topics contain "topic1", I call s.send(data, flags) synchronously. (Works)
- If topics contain "topic2", I call s.send(data, flags) asynchronously on subscribe but synchronously on publish. (Works)
- If topics contains "topic3" (non-working example). In case of subscribe, it is synchronous but async on publish. (Does not work)
- The problem is that calling s.send(...) asynchronously in publish doesn't seem to have any effect.

Asynchronously*: There is a http://google.com/ call with ngx.fetch(...), it awaits for the reply (then for the reply text), log the reply text, then call s.send(data, flags).