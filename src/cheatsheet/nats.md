# NATS

내가 정리한 [NATS](https://nats.io/) 메뉴얼

<https://github.com/djkoloski/rust_serialization_benchmark> data format serialize deseralize benchmark

<https://github.com/koute/speedy> 내가 사용할 메시지 포맷

<https://docs.rs/nats/latest/nats/jetstream/enum.AckKind.html> rust client

<https://docs.nats.io/nats-concepts/jetstream/consumers> consumer 사용법

<https://docs.nats.io/reference/faq#what-is-the-right-kind-of-stream-consumer-to-use>

<https://natsbyexample.com/examples/jetstream/pull-consumer/rust>

```bash
# 2023-08-08 기준 latest
# nats-server.conf file 자동 import 됌 
docker rm -f nats
docker run -d \
    --log-driver json-file --log-opt max-size=1g \
    --name=nats \
    --network host \
    -v ./nats-server.conf:/etc/nats/nats-server.conf \
    nats:2.9 -c /etc/nats/nats-server.conf
docker logs -f nats
```

```conf
# nats-server.conf file
server_name: "test_1"

# Client port of 4222 on all interfaces
port: 4222

# HTTP monitoring port
monitor_port: 8222

# This is for clustering multiple servers together.
cluster {
  # It is recommended to set a cluster name
  name: "c1"

  # Route connections to be received on any interface on port 6222
  port: 6222

  # Routes are protected, so need to use them with --routes flag
  # e.g. --routes=nats-route://ruser:T0pS3cr3t@otherdockerhost:6222
    authorization {        
        user: test 
        password: test-test-test
        timeout: 2
    }    


  # Routes are actively solicited and connected to from this server.
  # This Docker image has none by default, but you can pass a
  # flag to the nats-server docker image to create one to an existing server.
  routes = []
}

# 단일 노드 라서 에러남  Can't start JetStream: JetStream cluster requires configured routes or solicited leafnode for the system account
jetstream {        
    store_dir: /data
    max_mem: 1G
    max_file: 100G
}
``````
