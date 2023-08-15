# NATS

배포는 처음 실행기준 최신버전 사용 근데 이상한게 쉘 실행이 안됌 `ash`, `sh`, `bash`, `/bin/sh` 등등 다 해봤는데 안됌 근데 cli 따로 설치해서 사용할꺼니 상관없음.

```bash
# 2023-08-08 기준 latest
# config file 없이 실행 옵션으로 설정
docker rm -f nats
docker run -d \
    --log-driver json-file --log-opt max-size=1g \
    --name=nats \
    --network host \
    nats:2.9 --port 4222 --http_port 8222 --jetstream --store_dir /data --user test --pass test-test-test
docker logs -f nats
```

## CLI 노트

```bash
brew tap nats-io/nats-tools
brew install nats-io/nats-tools/nats

nats context add nats --server ip:port --user user_id --password user_pwd --description "NATS Demo" --select
nats stream info stream_name
nats stream subjects stream_name
```

- <https://github.com/djkoloski/rust_serialization_benchmark> data format serialize deseralize benchmark
- <https://github.com/koute/speedy> 내가 사용할 메시지 포맷
- <https://docs.rs/nats/latest/nats/jetstream/enum.AckKind.html> rust client
- <https://docs.nats.io/nats-concepts/jetstream/consumers> consumer 사용법
- <https://docs.nats.io/reference/faq#what-is-the-right-kind-of-stream-consumer-to-use>
- <https://natsbyexample.com/examples/jetstream/pull-consumer/rust>

## job queue

[scalable scheduling](/post/1.html) 관련 NATS 기능 찾아보는중

consumer가 Nak에 duration을 주고 응답하면 알아서 주어진 시간 이후로 message가 re-deilivery 되는거 같음 이러면 미래 반복 작업 처리 하기 훨씬 수월한데 아직 테스트는 안해봄 <https://docs.rs/async-nats/latest/async_nats/jetstream/message/enum.AckKind.html#variant.Nak>
