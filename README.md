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
  
+ Docker Swarm 설정

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