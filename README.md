# VATECH Junhyun
------------------
## Kafka cluster local 환경에 구성하기
kafka cluster는 보통 kafka(broker) 여러 대와 zookeeper 여러대로 구성된다. 물론 단일 kafka, zookeeper로 구성할 수 있다.

테스트를 위해 docker 환경에서 먼저 kafka cluster를 구성해보았다.

docker에 여러 컨테이너를 올리기에 용이한 docker-compose 방법을 썼다.

[단일 kafka, 단일 zookeeper 구성 yaml 파일](https://github.com/JackCokebb/kafka-all/blob/master/kafkaServer/docker-compose-lone.yml)

docker가 설치되어있다는 가정하에, docker를 작동시키고, terminal에서 코드를 실행시킨다.
``` 
//f: 파일명 지정, -d : background 실행
docker-compose -f docker-compose-lone.yml up -d
```

docker 프로그램으로 확인해보면 container들이 생성된 것을 확인할 수 있다.

--------------
### produce, comsume 확인
kafka에서 제공하는 shell 파일 내에 [kafka-console-consumer.sh, kafka-console-consumer.sh](https://kafka.apache.org/quickstart)로도 kafka 정상 작동 여부를 확인할 수 있지만,
[python code](https://github.com/2021-Vatech-skku/vatech/tree/junhyun/kafkaClients)를 이용해서 확인할 수 있다.

vscode와 같은 코드 에디터에서 실행해도 되고, terminal을 이용해도 된다.

python 3.x 이상 버전을 사용했다.
```bash
//python producer로 data를 kafka에 전송
python3 sample-producer.py

//python consumer로 data consume test
python3 sample-consumer.py

//실행시 port number, topic name, bootstrap.server 설정 등에 주의한다. 
//개인 설정에 맞게 실행
```
직전에 올린 docker-compose용 [단일 kafka, 단일 zookeeper 구성 yaml 파일](https://github.com/JackCokebb/kafka-all/blob/master/kafkaServer/docker-compose-lone.yml)에는[kafdrop](https://github.com/obsidiandynamics/kafdrop)이라는 kafka 모니터링 웹 UI 서비스도 같이 포함되어 있다.

yaml파일에서 지정해준 주소와 포트로 접속하면, kafka broker의 상태, topic list, message등 확인할 수 있다.

현재 docker-compose를 local환경에서 올렸다고 가정하고, yaml파일 기준 포트가 9001번으로 할당되어 있으므로, [localhost:9001](localhost:9001)로 접속해보면 kafka를 모니터링할 수 있다.

이제 python code로 생성한 message, topic등을 확인해볼 수 있다.

--------
## Deploying a Kafka cluster on a Kubernetes

kubernetes가 관리하는 cluster 내에 kafka를 올리기 위해서, [strimzi](https://strimzi.io/docs/operators/latest/using.html)에서 제공하는 operator와 custom resource 파일을 활용했다. 


### Prerequisites
[strimzi](https://strimzi.io/docs/operators/latest/deploying.html)에서 
+ Kubernetes 1.16 이후 버전의 cluster
+ kubectl command-line tool이 설치되고, kubernetes running cluster와 연결되어 있어야함

을 명시하고 있다.

#### namespace 생성
먼저 kubernetes에 kafka를 배치할 name space를 만들어 준다.

이때 namespace는 한 개여도 되고, 여러 개여도 상관없다.
```
//namespace creation example
kubctl create ns my-kafka-namespace

```

#### strimzi download
미리 [strimzi에서 release하는 최신 버전](https://github.com/strimzi/strimzi-kafka-operator/releases/)을 확인하자!
```
//download strimzi 0.24.0 ver
curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.24.0/strimzi-0.24.0.tar.gz -o strimzi.0.24.0.tar.gz

//unzip & extract tar file
tar -xvzf strimzi-0.24.0.tar.gz
```

### single namespace로 배치
-----
strimzi file에서 kubernetes namespace를 지정해주는 부분을 본인이 만든 namespace로 변경해주어야한다.

이 작업 이후에도, kubernetes위에 올리기 전에 항상 namespace를 잘 확인해야한다.

```
//on linux
sed -i 's/namespace: .*/namespace: my-kafka-namespace/' install/cluster-operator/*RoleBinding*.yaml

// on macOS 
sed -i '' 's/namespace: .*/namespace: my-kafka-namespace/' install/cluster-operator/*RoleBinding*.yaml
```

### Deploy the Cluster Operator
Cluster [Operator](https://www.redhat.com/ko/topics/containers/what-is-a-kubernetes-operator) 는 Custom Resource를 기반으로 kafka cluster 뿐만아니라 kafka connect, kafka user cluster도 생성해준다.

```
// move to downloaded strimzi file 
cd path/to/strimzi-0.24.0/

//deploy the cluster operator
kubectl create -f ./install/cluster-operator -n my-kafka-namespace
//-n : specify the namespace

//cluster operator가 잘 배치되었는지 확인 
kubectl get deployments -n my-kafka-namespace
```

### kafka cluster 배치
kafka 리소스를 만들기 위해 yaml파일을 이용해 특정 설정값들을 제공한다.

이 과정에서는 strimzi가 제공한 example YAMLs file을 이용했다.  strimzi.0.24.0/examples/ 에 있는 파일들이다.

kafka cluster 관련 yaml 파일들은 strimzi.0.24.0/examples/kafka 에 있다.  여러 대의 kafka를 사용할지, 어떤 종류의 cluster를 사용할지 정해서 배치하면 된다. cluster 종류는 [strimzi document](https://strimzi.io/docs/operators/latest/deploying.html#deploying-cluster-operator-str)를 참고하자.  

기본적으로 yaml에 topicOperator와 userOperator 설정이 되어있다. 
```
...
entityOperator
  topicOperator: {}
  userOperator: {}
  ```
  이런식으로 작성되어 있으면, 사용하겠다고 설정해둔 것.


kafka-ephemeral cluster 생성 - kafka : 3, zookeeper : 3, topicOperator: enabled, userOperator: enabled
```
kuberctl apply -f examples/kafka/kafka-ephemeral.yaml
```

배치 확인
```
kubectl get deployments -n my-kafka-namespace
```
------
## Kafka Connect
[kafka connect](https://docs.confluent.io/platform/current/connect/index.html)는 kafka cluster와 mongoDB, minio와 같은 다른 서비스들을 쉽게 연결시켜주는 tool이다.  connect cluster를 생성하고, cluster 내부에 각 서비스에 맞는connector 객체를 생성하는 방식이다.  따라서 connect cluster를 kubernetes에 배치할때 사용하는 image에, 각 서비스에서 제공하는 connector plugin들을 포함해야한다.
strimzi에서 제공하는 examples/connect/kafka-connect.yaml을 참고하여 [세로운 yaml]()을 만들었다.

