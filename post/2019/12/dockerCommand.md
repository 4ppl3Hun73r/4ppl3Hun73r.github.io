# 내가 자주 사용하는 Docker Command 정리

## docker ps -a
docker container 보기

## docker exec -it {container ID} {exec command}
docker 컨테이너에 명령어 실행
```
$ docker exec -it c903213b1 /bin/bash
```

## docker logs {container ID}
docker 컨테이너 로그 확인

### 컨테이너 로그 위치
/var/lib/docker/containers/{container ID}/{container Name}-json.log

### 컨테이너 로그 지우기
```
$ truncate -s 0 /var/lib/docker/containers/*/*-json.log
```
