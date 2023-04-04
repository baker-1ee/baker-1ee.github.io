지난 시간에 이어서 오늘은 DB 구성을 진행하였다. 

DB는 MySQL 이며, Docker 설치 후 docker compose 파일을 작성하여 컨테이너 상에서 실행시키도록 구성하였다.

docker compose 파일은 프로젝트 디렉토리 하위에 docker 디렉토리를 생성하여 작성하였다.
> docker/docker-compose.yml

도커로 DB 를 구성하면 아래와 같이 간단하게 컨테이너만 실행 or 중지 할 수 있어서 좋다.

<img src="assets/images/docker.png" alt="docker">