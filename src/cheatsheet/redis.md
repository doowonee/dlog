# Redis

공식문서인 <https://redis.io/documentation>를 참고하고 공식 도커 이미지인 <https://hub.docker.com/_/redis>를 사용한다. 버전은 현재 시점 기준 최신 안정 버전인 `6.2.4` 버전을 사용한다.

```bash
# 기본 설정 그대로 사용
docker run --name redis -d \
    --network=host \
    --log-driver json-file --log-opt max-size=3g \
    redis:6.2.4
```

<https://redis.io/topics/config#configuring-redis-as-a-cache> Redis를 LRU 캐시로만 사용하려면 아예 키에 해당하는 데이터 크기와 모든 키의 expire set을 지정할수 있다.

Redis를 위한 OS 설정은 <https://redis.io/topics/admin>를 참고 하면된다. `overcommit_memory` 등 해야할게 은근 있다.

Replica 설정은 기본적으로 RDB save 기능 즉 disk io를 사용한다. swap을 사용하라고 함 안하면 OS OOM killer가 프로세스 죽일수도 있다고 한다.

`CONFIG SET`또는 `CONFIG REWRITE`로 웬만한 설정을 재기동없이 변경할수 있는것으로 보인다.

swap없이 기본 설정 변경없이 실행하면 redis가 사용하는 최대 메모리는 0 즉 무한이다.

우분투는 `sudo apt install install redis-tools`로 `redis-cli` 툴을 사용할수 있다.

## CLI 노트

```bash
# 메모리 설정 보기
config get maxmemory

# redis 클러스터 모드로 연결
redis-cli -c -p 7001
cluster nodes

# 싱글 서버 용도로 클러스터일 경우 각 노드에서 실행 해야 한다
# 실행 결과도 각 노드 별로 다르다
SCAN 0 MATCH users@* COUNT 100
# 아래 방식은 사용하지 말라고함 근데 이것도 어짜피 노드별로 결과가 다르다
KEYS users*

# 키 삭제 및 조회
# 연결이 포워딩 되서 지워지긴 함 지워지고 redis-cli 프롬프트에 서버 주소랑 포트가 달라져있으며
# 키 스캔 결과도 그에 따라 달라지게 된다
DEL users@214214215125
GET users@346373473473

# 알아서 내부적으로 scan 호출해서 전부 탐색함 -i 은 인터벌로 서버 조회에 대한 서버 부하 감소시킴
redis-cli -c -p 7001 -i 0.01 --scan --pattern 'sub*'

# 탐색결과로 나온 값 전부 삭제 하기 근데 나중에 수만 수십만개면 어떻하지 이거?
# unlink는 메모리 할당 해제는 나중에 비동기로 하고 키 영역만 제거 하는걸로 이거 더 서버 부하가 적음 DEL은 동기임
redis-cli -c -p 7001 -i 0.01 --scan --pattern 'users@*' | xargs -n 1 -d '\n' redis-cli -c -p 7001 UNLINK
redis-cli -c -p 6379 --scan --pattern "users*" | xargs -n 1 redis-cli -c -p 6379 DEL
```
