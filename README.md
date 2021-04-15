#### Docker Swarm
+ What is Docker Swarm?
    + 여러 컨테이너를 클러스터로 만들어 관리
    + Docker를 사용한 orchestration infra를 구축할 때 가장 호환성이 좋음

![Docker Swarm](https://github.com/suhojang/dockerswram/blob/master/docker-swram.png)

+ Node
  + 클러스터에 속한 도커 서버 단위 입니다. 보통 한 서버에 하나의 도커 데몬을 실행하기 때문에 노드는 곧 서버라고 이해 할 수 있습니다.   
  
+ Manager
  + 매니저는 클러스터의 상태를 관리합니다. 명령어는 매니저 노드에서만 실행 할 수 있습니다.   
    아키텍쳐상에서 매니저는 High Avalilability를 위하여 여러대 실행 되어야 합니다.   
    일반적으로 노드마다 매니저가 배포 됩니다.   
 
+ Worker
  + 매니저의 명령을 받아 컨테이너를 생성하고 상태를 체크합니다.   
  서비스 규모에 맞게 많이 실행하고 요청이 많아지면 Worker를 스케일아웃 합니다.
    
+ Service Discovery
  + 서비스 디스커버리는 컨테이너가 실행 위치와 상태를 제공 해 줍니다.   
  이를 위하여 자체 DNS 서버를 가지고 있습니다.   
    컨테이너를 생성하면 서비스명과 동일한 도메인을 등록하고 반대로 멈추면 도메인을 제거 합니다.   
    Consul, etcd, zookeeper와 같은 외부 서비스를 사용하지 않아도 되고 Swarm이 내부에서 자체적으로 처리해 줍니다.   
    OpenStack의 Keystone과 AWS의 IAM과 비교할 수 있습니다.   
    
+ Service
  + 기본적으로 배포 단위 입니다.    
    하나의 서비스는 하나의 이미지를 기반으로 생성하고 동일한 컨테이너를 한개 이상 실핼 할 수 있습니다.   
    최종적으로 배포 되는 서비스는 여러 개의 Task로 구성 됩니다.   
    
+ Task
  + 컨테이너 배포 단위 입니다. 각각의 Task가 컨테이너를 관리합니다.   
  보통 개별 도커 컨테이너를 의미하지만 컨테이너를 실행할 때 명령어도 포함 합니다.   
  

+ manager와 worker간 join을 위한 token 생성
  + manager에서 아래와 같이 명령어 실행
```groovy
여기서 docker01은 manager docker02는 worker
```  

```shell
[root@docker01 swarm]# docker swarm init
Swarm initialized: current node (eudk8re3rg8h8gja99tj1c9ll) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3lbhc8lt1h52jcmkvdr464fnn05zf3t8urecjqy32rn3r7y2ky-ek7abd9537zsx9wfje1acswif \
    192.168.137.2:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

[root@docker01 swarm]#
```
+ manager와 join을 위해 위에 생성 된 join 명령문을 실행
```shell
[root@docker02 ~]# docker swarm join \
>     --token SWMTKN-1-1kvu8z8t5dxx6bile1qvbi1qxm0htp8we89yuiocna077t2d8u-72014ptnje3267comxjrg24cg \
>     192.168.137.2:2377
This node joined a swarm as a worker.
[root@docker02 ~]#
```

+ Test를 위한 yml파일 생성
```yaml
version: "3"

networks:
  service-network

#Container 생성
services:
  web:
    image: mousai86/hostname:latest
    ports:
      - "80:80"
      networks:
        - service-network
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.1"
          memory: 20M
      restart_policy:
        condition: on-failure
        delay: 2s
        max_attempts: 3

# services.web.deploy.replicas = 생성 할 container 개수 지정
# services.web.deploy.resources.limits.cpus = container의 cpu 사용량 지정
# services.web.deploy.resources.limits.memory = container의 memory 사용량 지정
# services.web.deploy.restart_policy.condition = 종류[none(설정하지 않음), on-failure(실패 시), any(항상)기본값]
# services.web.deploy.restart_policy.delay = 체크할 시간 지정
# services.web.deploy.restart_policy.max_attempts = 시도 할 횟수
```

+ docker-compose를 이용하여 service 및 node 생성
```shell
[root@docker01 swarm]# docker stack deploy -c docker-compose.yml hostname
Creating network hostname_service-network
Creating service hostname_web
[root@docker01 swarm]#
```

+ docker service가 생성 되었는지 확인
```shell
[root@docker01 swarm]# docker stack ls
NAME      SERVICES
hostname  1

[root@docker01 swarm]# docker service ls
ID            NAME          MODE        REPLICAS  IMAGE
a4pk7xq9xetr  hostname_web  replicated  3/3       mousai86/hostname:latest
[root@docker01 swarm]#
```

+ docker service Name으로 process 확인
```shell
[root@docker01 swarm]# docker service ps hostname_web
ID            NAME            IMAGE                     NODE      DESIRED STATE  CURRENT STATE          ERROR  PORTS
w7348rsr1r99  hostname_web.1  mousai86/hostname:latest  kube-m    Running        Running 2 minutes ago
zu686sh6tdeh  hostname_web.2  mousai86/hostname:latest  docker02  Running        Running 2 minutes ago
n427wjv5dgzr  hostname_web.3  mousai86/hostname:latest  kube-m    Running        Running 2 minutes ago
```

+ docker container 확인
```shell
[root@docker01 swarm]# docker ps
CONTAINER ID        IMAGE                                                                                       COMMAND             CREATED             STATUS              PORTS               NAMES
bfae3ea981e0        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     4 minutes ago       Up 4 minutes        80/tcp              hostname_web.3.n427wjv5dgzryt8sjej888fpr
e7e1216a8689        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     4 minutes ago       Up 4 minutes        80/tcp              hostname_web.1.w7348rsr1r99wyvor0shp0sf3

[root@docker02 ~]# docker ps
CONTAINER ID        IMAGE                                                                                       COMMAND             CREATED             STATUS              PORTS               NAMES
1c00365ee445        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     5 minutes ago       Up 5 minutes        80/tcp              hostname_web.2.zu686sh6tdehh739tj1f61sfl
```

+ yml에 설정 한  services.web.deploy.restart_policy 설정이 작동 되는지 확인
  + docker의 container를 stop하면 자동으로 container가 manager에 생성 된 걸 확인 할 수 있다.
  + docker service ps [서비스명]을 이용하여 shutdown/Running Time을 확인할 수 있다.
```shell
[root@docker02 ~]# docker ps
CONTAINER ID        IMAGE                                                                                       COMMAND             CREATED             STATUS              PORTS               NAMES
fb26cd490f29        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     3 minutes ago       Up 3 minutes        80/tcp              hostname_web.3.jm3sb64rf8nvxu5blkkdwha3y
9f7ce97bba5b        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     3 minutes ago       Up 3 minutes        80/tcp              hostname_web.2.exynhrbcn7ak65syeq403hyjn
92ca2502741d        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     3 minutes ago       Up 3 minutes        80/tcp              hostname_web.1.xal4qnkcerenqxxrdj99u63fi
[root@docker02 ~]# docker stop hostname_web.3.jm3sb64rf8nvxu5blkkdwha3y
hostname_web.3.jm3sb64rf8nvxu5blkkdwha3y
[root@docker02 ~]# docker ps
CONTAINER ID        IMAGE                                                                                       COMMAND             CREATED             STATUS              PORTS               NAMES
9f7ce97bba5b        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     4 minutes ago       Up 3 minutes        80/tcp              hostname_web.2.exynhrbcn7ak65syeq403hyjn
92ca2502741d        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     4 minutes ago       Up 4 minutes        80/tcp              hostname_web.1.xal4qnkcerenqxxrdj99u63fi

[root@docker01 swarm]# docker ps
CONTAINER ID        IMAGE                                                                                       COMMAND             CREATED             STATUS              PORTS               NAMES
2e63329f9dad        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     31 seconds ago      Up 28 seconds       80/tcp              hostname_web.3.mxp0i1tujwehq7fh4p5xku84v
[root@docker01 swarm]# docker service ps hostname_web
ID            NAME                IMAGE                     NODE      DESIRED STATE  CURRENT STATE           ERROR                        PORTS
xal4qnkceren  hostname_web.1      mousai86/hostname:latest  docker02  Running        Running 4 minutes ago
w7348rsr1r99   \_ hostname_web.1  mousai86/hostname:latest  kube-m    Shutdown       Failed 7 minutes ago    "task: non-zero exit (137)"
exynhrbcn7ak  hostname_web.2      mousai86/hostname:latest  docker02  Running        Running 4 minutes ago
zu686sh6tdeh   \_ hostname_web.2  mousai86/hostname:latest  docker02  Shutdown       Shutdown 4 minutes ago
mxp0i1tujweh  hostname_web.3      mousai86/hostname:latest  kube-m    Running        Running 44 seconds ago
jm3sb64rf8nv   \_ hostname_web.3  mousai86/hostname:latest  docker02  Shutdown       Failed 47 seconds ago   "task: non-zero exit (137)"
n427wjv5dgzr   \_ hostname_web.3  mousai86/hostname:latest  kube-m    Shutdown       Shutdown 4 minutes ago
[root@docker01 swarm]#
```

+ docker command로 scale 조절하기
```shell
[root@docker01 swarm]# docker service scale hostname_web=1
hostname_web scaled to 1

[root@docker01 swarm]# docker service ls
ID            NAME          MODE        REPLICAS  IMAGE
a4pk7xq9xetr  hostname_web  replicated  1/1       mousai86/hostname:latest
[root@docker01 swarm]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

[root@docker02 ~]# docker ps
CONTAINER ID        IMAGE                                                                                       COMMAND             CREATED             STATUS              PORTS               NAMES
9f7ce97bba5b        mousai86/hostname@sha256:49cf580ab47e837e710baa3af661941fdf260960756b163972376a65367776c9   "python app.py"     9 minutes ago       Up 8 minutes        80/tcp              hostname_web.2.exynhrbcn7ak65syeq403hyjn

[root@docker01 swarm]# docker service scale hostname_web=5
hostname_web scaled to 5
[root@docker01 swarm]# docker service ls
ID            NAME          MODE        REPLICAS  IMAGE
a4pk7xq9xetr  hostname_web  replicated  5/5       mousai86/hostname:latest
[root@docker01 swarm]# docker service ps hostname_web
ID            NAME                IMAGE                     NODE      DESIRED STATE  CURRENT STATE            ERROR  PORTS
xv482i7pc0w7  hostname_web.1      mousai86/hostname:latest  kube-m    Running        Running 8 seconds ago
exynhrbcn7ak  hostname_web.2      mousai86/hostname:latest  docker02  Running        Running 11 minutes ago
zu686sh6tdeh   \_ hostname_web.2  mousai86/hostname:latest  docker02  Shutdown       Shutdown 11 minutes ago
mwlb2siq8fgb  hostname_web.3      mousai86/hostname:latest  kube-m    Running        Running 7 seconds ago
shydp8x3qx1j  hostname_web.4      mousai86/hostname:latest  docker02  Running        Running 9 seconds ago
mm4y3i052sf2  hostname_web.5      mousai86/hostname:latest  kube-m    Running        Running 7 seconds ago
```

+ ETC
```shell
# docker service 를 update 시키기
[root@docker01 swarm]# docker service update hostname_web
hostname_web
# docker service 를 강제로 update 시키기
[root@docker01 swarm]# docker service update hostname_web --force
hostname_web
```