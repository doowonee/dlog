# Docker

```bash
# 300시간 보다 오래된 사용하지 않는 도커 이미지를 삭제한다
docker image prune -a --filter "until=300h"

# 컨테이너 로그 파일 위치만 출력
docker inspect --format='{{.LogPath}}' container_id

# 특정 컨테이너들 한번에 제거하기
docker rm -f  $(docker ps -a --format '{{.Names}}' | grep name_what_i_want)

# 모든 컨테이너 로그 파일 사이즈 출력
sudo du -h $(docker inspect --format='{{.LogPath}}' $(docker ps -qa))
```

## Docker registry

docker custom registry 쓰면 인증 서버 재부팅 해도 인증 정보 ~/.docker/config.json 정보는 expired 안된다 그대로 사용 가능함 그냥 비밀번호를 암호화해서 파일로 저장하는거라함

## ENTRYPOINT VS CMD

`ENTRYPOINT`는 항상 실행되어야할 명령어이고 CMD는 인자라고 보면된다. 즉 아래와 같이 되어있을때 `docker run -it akamaiapi` 실행시 `node src/akamai-client.js purge https://www.example.com/main.css`가 실행되고 `docker run -it akamaiapi test` 입력시 `node src/akamai-client.js test`가 실행 된다.

```bash
ENTRYPOINT ["node", "src/akamai-client.js"]
CMD ["purge", "https://www.example.com/main.css"]
```

## Docker compose

`docker-compose` 는 버전 1이고 `docker compose`가 버전 2임 superset을 예로 들면 `docker compose -f docker-compose-non-dev.yml rm -f` 이런식으로 특정 compose 파일 기준 컨테이너 삭제 등의 작업을 수행 할수 있음 `docker compose -f docker-compose-non-dev.yml down -v`로 모든 컨테이너 제거 및 볼륨도 제거 할수 있음
