# Stage 2. Stage1 + distributed processing using apache spark
- logstash에서  kafka로 저장하고, 이를 spark에서 실시간 분산처리 후 ES에 저장
 * logstash > kafka > spark streaming > ES/redis

## Stage 2의 주요 내용
### 1 Stage1의 한계
 * Stage1에서는 실시간 Data가 많아지게 될 경우, 하나의 logstash로는 대량의 데이터 처리가 어려운 상황이다.
 * 또한 customer_id, track_id 이외의 구체적인 정보가 없어서 세분화된 분석을 하기 어렵다 (예를 들면 남성이 가장 좋아하는 음악은?)
 * 매번 ES전체 table을 조회하여 데이터를 시각화하게 되어, 성능상의 부하가 예상된다.

### - Technical changes (support huge data processing using spark)
 * logstash의 biz logic(filter)을 단순화하여 최대한 많은 양을 전송하는 용도로 활용한다.
 * 그리고 kafka를 이용하여 대량의 데이터를 빠르고, 안전하게 저장 및 전달하는 Message queue로 활용한다.
 * Spark streaming은 kafka에서 받아온 데이터를 실시간 분산처리하여 대상 DB(ES or others)에 병렬로 저장한다.
  - 필요한 통계정보(최근 30분간 접속통계 등을 5분단위로 저장 등) 및  복잡한 biz logic지원
 * redis는 spark streaming에서 customer/music id를 빠르게 join하기 위한 memory cache역할을 한다.

### - Software 구성도
![stage2 architecture](https://github.com/freepsw/demo-spark-analytics/blob/master/resources/images/stage2.png)

## [STEP 1] Install the elastic stack (Stage01 내용 참고)
- Stage01 내용을 참고하여 elsaticsearch, logstash, kibana를 설치 및 실행



## [STEP 2] Install & start kafka [link](http://kafka.apache.org/documentation.html#quickstart)
### Download the apache kafka binary files
```
> sudo yum install -y wget
> cd ~/demo-spark-analytics/sw
> wget https://archive.apache.org/dist/kafka/3.0.0/kafka_2.12-3.0.0.tgz
> tar xvf kafka_2.12-3.0.0.tgz
```

### Start Zookeeper server
```
> cd ~/demo-spark-analytics/sw/kafka_2.12-3.0.0

# 1) Foreground 실행 (테스트 용으로 zookeeper 로그를 직접 확인)
> bin/zookeeper-server-start.sh config/zookeeper.properties

# 2) Background 실행
> bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
> ps -ef | grep zookeeper
```

### Start Kafka Broker 
-  kafka delete option 설정
  - topic을 삭제할 수 있는 옵션 추가 (운영서버에서는 이 옵션을 false로 설정. topic을 임의로 삭제할 수 없도록)
```
> cd ~/demo-spark-analytics/sw/kafka_2.12-3.0.0
# 1) Foregroud 
> bin/kafka-server-start.sh config/server.properties

# 2) background 실행
> bin/kafka-server-start.sh -daemon config/server.properties

```
#### Kafka Broker 설치후 생성되는 로그파일
- config/server.properties의 log.dir에 정의된 파일
  - https://github.com/apache/kafka/blob/3.0/config/server.properties
```
## Broker 관련 메타 정보를 출력 
> cat /tmp/kafka-logs/meta.properties
#
#Sun Mar 27 00:10:12 UTC 2022
cluster.id=OX9OhkJmRZWJ9xFCE8CIgw
version=0
broker.id=0
```

### Create a topic
#### topic 생성 후 조회 
```
> cd ~/demo-spark-analytics/sw/kafka_2.12-3.0.0
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic realtime

> bin/kafka-topics.sh --list --bootstrap-server localhost:9092
realtime
```
- replication-factor : 메세지를 복제할 개수 (1은 원본만 유지)
- partitions : 메세지를 몇개로 분산하여 저장할 것인지 결정 (갯수 만큼 병렬로 write/read 함)


### Test Apache kafka 
- Create a topic "test" and send message using kafka producer
```
> cd ~/demo-spark-analytics/sw/kafka_2.12-3.0.0
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test

> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
my message
second message
```

- Receive the messages from topic "test" using consumer 
```
> cd ~/demo-spark-analytics/sw/kafka_2.12-3.0.0
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
my message
second message
```

## [STEP 3] Install & start redis (redis 3.0.7)
```
> cd ~/demo-spark-analytics/sw
> wget http://download.redis.io/releases/redis-3.0.7.tar.gz
> tar -xzf redis-3.0.7.tar.gz
> cd redis-3.0.7
> sudo yum -y install gcc-c++ make
> make
```
- "zmalloc.h:51:31: error: jemalloc/jemalloc.h: No such file or directory"에러 발생시
```
> make distclean
> make
```

### Start redis server 
```
> cd ~/demo-spark-analytics/sw/redis-3.0.7
> src/redis-server redis.conf
```

#### Test redis 
```
> cd ~/demo-spark-analytics/sw/redis-3.0.7
> src/redis-cli
redis> set user_id 001
OK
redis> get user_id
"001"
```

## [STEP 4] Install & start apahche spark (spark-2.4.8-bin-hadoop2.7)
```
> cd ~/demo-spark-analytics/sw/
> wget https://archive.apache.org/dist/spark/spark-2.4.8/spark-2.4.8-bin-hadoop2.7.tgz
> tar xvf spark-2.4.8-bin-hadoop2.7.tgz
> cd spark-2.4.8-bin-hadoop2.7
```

### Set spark configuration
- spark environment
```
# slave 설정
> cp conf/slaves.template conf/slaves
localhost //현재  별도의 slave node가 없으므로 localhost를 slave node로 사용

# spark master 설정
# 현재 demo에서는 별도로 변경할 설정이 없다. (실제 적용시 다양한 설정 값 적용)
> cp conf/spark-env.sh.template conf/spark-env.sh
> cp conf/log4j.properties.template conf/log4j.properties
```

- add spark path to system path
```
> vi ~/.bash_profile
# 마지막 line에 아래 내용을 추가한다.
export SPARK_HOME=~/demo-spark-analytics/sw/spark-2.4.8-bin-hadoop2.7
export PATH=$PATH:$SPARK_HOME/bin

> source ~/.bash_profile
```

### Set ssh connection without password
```
> ssh-keygen -t rsa

## 생성된 key (public, private) 확인 
> ls ~/.ssh
authorized_keys  id_rsa  id_rsa.pub

## 현재 인증된 ssh key 확인 (사용자계정@gmail.com으로 생성된 키만 존재)
> cat ~/.ssh/authorized_keys
# Added by Google
ssh-rsa AAAAB3NzaC1yc2EAAAADAQ........TmGP5rBf7KAtNtHL0= freeeeee@gmail.com

## 방금 생성한 public key도 등록하여, password 없이 접속 가능하도록 설정 
> cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

## ssh public key가 정상 등록되었는지 확인 
> cat ~/.ssh/authorized_keys
# Added by Google
ssh-rsa AAAAB3NzaC1yc2EAAAADAQ........TmGP5rBf7KAtNtHL0= freeeeee@gmail.com
ssh-rsa AAAAB3NzaC1yc2EAAAADAQ........Uo980M4Y/qclAxRsWb freeeeee@demo-test

> chmod og-wx ~/.ssh/authorized_keys 

# 정상적으로 접속되는지 확인
> ssh user_id@localhost
```


### Start spark master
```
> cd ~/demo-spark-analytics/sw/spark-2.4.8-bin-hadoop2.7
> sbin/start-all.sh
```

### Open spark master web-ui with web browser
localhsot:8080

#### [Error 1] Permission Error 발생시
- 아래와 worker가 정상적으로 등록되지 않았다면,
- ssh connection이 localhost에 정상적으로 접속하지 못해서 생기는 문제이다.
- ssh connection을 자동으로 접속할 수 있도록 하자.
```
# 1) ssh key 생성
> ssh-keygen -t rsa

# 2-1) 외부 ssh연결을 할 경우, ssh public key를 remote서버로 복사
#    remote server의  ~/.ssh/authorized_keys에 추가된다.
> ssh-copy-id <계정명>@ip

# authorized_keys 파일의 권한이 부여되지 않아서 발생함.
# 어떤 원인으로 해당 권한이 달라졌는지는 알수 없으나, 아래와 같이 권한 부여
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/authorized_keys

# spark 재시작
> cd ~/demo-spark-analytics/sw/spark-2.4.8-bin-hadoop2.7
> sbin/stop-all.sh

> sbin/start-all.sh
```


## [STEP 5] Import customer info to redis

### Install python redis package (python 3.6)
```
> cd ~
> sudo yum install -y https://repo.ius.io/ius-release-el7.rpm
> sudo yum install -y python36u python36u-libs python36u-devel python36u-pip
> python3 -V
Python 3.6.8
> pip3 -V
pip 9.0.3 from /usr/lib/python3.6/site-packages (python 3.6)
```

#### Create python virtual env
```
> cd ~/demo-spark-analytics
> sudo pip3 install virtualenv

## python 3.6 가상환경 생성 
> virtualenv venv

## 생성된 python3.6 가상환경 정보 확인 
> ls venv
bin  lib  lib64  pyvenv.cfg

> ls venv/bin
activate       activate.nu       deactivate.nu  pip3    python3    wheel-3.6
activate.csh   activate.ps1      pip            pip3.6  python3.6  wheel3
activate.fish  activate_this.py  pip-3.6        python  wheel      wheel3.6

## 현재 python 환경 확인 (2.7 버전)
> python -V
Python 2.7.5

## 생성한 python 가상환경으로 변경 (3.6 버전 환경으로 변경됨)
> source venv/bin/activate

(venv)> python -V
Python 3.6.8

(venv)> pip install redis
(venv)> pip install numpy
```


### Run import_customer_info.py (read customer info and insert into redis)
```
(venv)> cd ~/demo-spark-analytics/00.stage2
(venv)> python import_customer_info.py

## 현재 가상환경 종료하기 
(venv)> deactivate
```
- redis에 정상적으로 저장되었는지 확인
```
> cd ~/demo-spark-analytics/sw/redis-3.0.7
> src/redis-cli
# 전체 키를 조회하는 명령어
redis> keys *
# 아래와 같이 사용자 id별로 key가 생성된 것을 볼 수 있다.
4995) "2567"
4996) "3623"
4997) "1996"
4998) "2638"
4999) "4256"
```

### import_customer_info.py code
```javascript
import redis
import csv
import numpy as np
import random

# 사용자에게 임의의 나이를 부여한다.
# return age : 10세 단위의 나이에서 특정 나이 (11, 14..)를 랜덤으로 추출
def get_age():
    # age10 : 10, 20..과 같은 나이대
    # 나이대 별로 가중치를 부여하여, 20가 가장 많이 선택되고,
    # 그 다음으로 30, 40.. 으로 추출 되도록 설정
    age10 = random.choice([10, 20, 20, 20, 20, 30, 30, 30, 40, 40, 50, 60])
    start   = age10
    end     = age10 + 10
    # 선택된 나이대의 1자리 숫자도 랜덤으로 부여
    age = random.choice(np.arange(start, end))
    return age

# 1. redis server에 접속한다.
r_server = redis.Redis('localhost')

# 2. read customer info from file(csv)
with open('./cust.csv', 'rt') as csvfile:
    # csv 파일에서 고객정보를 읽어온다.
    reader = csv.DictReader(csvfile, delimiter = ',')
    next(reader, None)
    i = 1
    # save to redis as hashmap type
    # hash map 자료구조가 1개의 key에 다양한 사용자 정보를 저장/조회 할 수 있다.
    for row in reader:
      # 고객 id를 Key로 지정하고, name, gender, age, address..등 다양한 정보를 저장
        # db table로 비교하면, 1개의 레코드를 생성(pk는 custid, field는 name, gender....)
        r_server.hmset(row['CustID'], {'name': row['Name'], 'gender': int(row['Gender'])})
        r_server.hmset(row['CustID'], {'age': int(get_age())})
        r_server.hmset(row['CustID'], {'zip': row['zip'], 'Address': row['Address']})
        r_server.hmset(row['CustID'], {'SignDate': row['SignDate'], 'Status': row['Status']})
        r_server.hmset(row['CustID'], {'Level': row['Level'], 'Campaign': row['Campaign']})
        r_server.hmset(row['CustID'], {'LinkedWithApps': row['LinkedWithApps']})

        # 진행상황을 확인하기 위한 프린트...
        if(i % 100 == 0):
          print('insert %dth customer info.' % i)
        i += 1

    print('importing Completed (%d)' % i)
```

- 사용자 상세 정보가 정상적으로 redis에 저장되었는지 확인
```
> cd ~/demo-spark-analytics/sw/redis-3.0.7
> src/redis-cli
127.0.0.1:6379> hgetall 2 #사용자 id 2번에 대한 정보를 조회
 1) "gender"
 2) "0"
 3) "name"
 4) "Paula Peltier"
 5) "age"
 6) "30"
 7) "zip"
 8) "66216"
 9) "Address"
10) "10084 Easy Gate Bend"
11) "Status"
12) "1"
13) "SignDate"
14) "01/13/2013"
15) "Campaign"
16) "4"
17) "Level"
18) "0"
19) "LinkedWithApps"
20) "1"
```

## [STEP 6] Run logstash (read logs --> kafka)

### logstash configuration
- input
  - tracks_live.csv (stage1에서 data_generator로 생성하는 file을 활용)
- filter (적용하지 않음)
  - 이번 시나리오에서 logstash는 빠르게 수집하는 것에 초점.
  - 실제 데이터에 대한 처라(filter, aggregation...)는 spark streaming에서 처리
-  output
  - codec : logs에서 읽은 문자열을 그대로 kafka로 저장
  - bootstrap_servers : kafka server(broker)의 ip:port
  - topic_id : kafka에서 생성한 topic 명 (spark에서 동일 topic명으로 데이터를 읽음)

```javascript
> cd ~/demo-spark-analytics/00.stage2
> vi logstash_stage2.conf

input {
  file {
    path => "/home/본인계정으로변경/demo-spark-analytics/00.stage1/tracks_live.csv"
  }
}

output {
  stdout {
    codec => rubydebug{ }
  }

  kafka {
    codec => plain {
      format => "%{message}"
    }
    bootstrap_servers => "localhost:9092"
    topic_id => "realtime"
  }
}
```

### run logstash
```
> cd ~/demo-spark-analytics/00.stage2
> ~/demo-spark-analytics/sw/logstash-7.10.2/bin/logstash -f logstash_stage2.conf
# 아래와 같은 메세지가 출력되면 정상
Settings: Default pipeline workers: 8
Pipeline main started
{
               "message" => "0,48,453,\"2014-10-23 03:26:20\",0,\"72132\"",
              "@version" => "1",
            "@timestamp" => "2016-10-25T08:58:09.796Z",
                  "path" => "/Users/skiper/work/DevTools/github/demo-spark-analytics/00.stage1/tracks_live.csv",
                  "host" => "skcc11n00142",
              "event_id" => "0",
           "customer_id" => "48",
              "track_id" => "453",
              "datetime" => "2014-10-23 03:26:20",
              "ismobile" => "0",
    "listening_zip_code" => "72132"
}
.....
```

### Check received message from kafka using kafka-console_consumer
- logstash에서 kafka로 정상적으로 메세지가 전송되고 있는지 모니터링
- 아래의 kafka-console-consumer 명령어를 통해 전송되는 메세지를 확인
```
> cd ~/demo-spark-analytics/sw/kafka_2.12-3.0.0
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic realtime
# logstash에서 정상적으로 메세지를 보내면, 아래와 같은 메세지가 출력될 것임.
0,48,453,"2014-10-23 03:26:20",0,"72132"
1,1081,19,"2014-10-15 18:32:14",1,"17307"
2,532,36,"2014-12-10 15:33:16",1,"66216
```

## [STEP 7] run apache spark streaming application
### create spark application project using maven
#### - 참고 create scala/java project using maven [link](https://github.com/freepsw/java_scala)
- spark application은 scala와 java를 모두 사용할 수 있으므로,
- maven project 구성 시에 .java, .scala파일을 모두 인식할 수 있도록 설정해야 한다.
- [link](https://github.com/freepsw/java_scala) 프로젝트를 참고.

#### - pom.xml에 dependency 추가
- pom.xml (프로젝트에서 사용하는 library 설정)
- 여기에서 지정한 library는 mavend에서 자동으로 download하여 compile, run time에 참조한다.

```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka-0-10_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch-spark-20_2.11</artifactId>
            <version>7.10.1</version>
        </dependency>
        <dependency>
            <groupId>net.debasishg</groupId>
            <artifactId>redisclient_2.10</artifactId>
            <version>3.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
    </dependencies>
```

#### Spark streaming driver 코드 작성
-  SparkContex에 필요한 configuration을 설정한다.
-  StreamingContext를 생성 (위의 sparkcontext활용, batch주기는 2초)

-  [STEP 1]. Create Kafka Receiver and receive message from kafka broker
 * kafka에서 데이터를 받기위한 Kafka receiver를 생성한다.
 * 만얀 kafka partition이 여러개 일 경우, numReceiver를 partition 갯수만큼 지정
 * kafka receiver가 많아지면 데이터를 병렬로 읽어오게 된다.
 * union을 활용하여 1개의 rdd로 join한다. (논리적으로 1개로 묶였을 뿐, 내부적으로는 여러개의 partition으로 구성됨)

- [STEP 2]. parser message and join customer info from redis
 * kafkad에서 받아온 메세지 중에서 customer_id를 추출
 * customer_id를 key로 redis에서 사용자 상세 정보를 조회 (import_customer_info.py에서 저장한 고객정보)
 * kibana에서 사용할 timestamp field는 현재 시간으로 설정

- [STEP 3]. Write to ElasticSearch
 * kafka data + redis 고객정보를 합쳐서 elasticsearch에 저장
 * full source code [link](https://github.com/freepsw/demo-spark-analytics/blob/master/00.stage2/demo-streaming/src/main/scala/io/skiper/driver/Stage2StreamingDriver.scala)

```scala
object Stage2StreamingDriver {
  def main(args: Array[String]) {
    //[STEP 1] create spark streaming session
    // Create the context with a 1 second batch size
    val sparkConf = new SparkConf().setMaster("local[2]").setAppName("Stage2_Streaming")
    sparkConf.set("es.index.auto.create", "true");
    sparkConf.set("es.nodes", "localhost")
    val ssc = new StreamingContext(sparkConf, Seconds(2))
    addStreamListener(ssc)

    // [STEP 1]. Create Kafka Receiver and receive message from kafka broker
    val Array(zkQuorum, group, topics, numThreads) = Array("localhost:2181" ,"realtime-group1", "realtime", "2")
    ssc.checkpoint("checkpoint")
    val topicMap    = topics.split(",").map((_, numThreads.toInt)).toMap
    val numReceiver = 1

    // parallel receiver per partition
    val kafkaStreams = (1 to numReceiver).map{i =>
      KafkaUtils.createStream(ssc, zkQuorum, group, topicMap).map(_._2)
    }
    val lines = ssc.union(kafkaStreams)

    // [STEP 2]. parser message and join customer info from redis
    // original msg = ["event_id","customer_id","track_id","datetime","ismobile","listening_zip_code"]
    val columnList  = List("@timestamp", "customer_id","track_id","ismobile","listening_zip_code", "name", "age", "gender", "zip", "Address", "SignDate", "Status", "Level", "Campaign", "LinkedWithApps")
    val wordList    = lines.mapPartitions(iter => {
      val r = new RedisClient("localhost", 6379)
      iter.toList.map(s => {
        val listMap = new mutable.LinkedHashMap[String, Any]()
        val split   = s.split(",")

        listMap.put(columnList(0), getTimestamp()) //timestamp
        listMap.put(columnList(1), split(1).trim) //customer_id
        listMap.put(columnList(2), split(2).trim) //track_id
        listMap.put(columnList(3), split(4).trim.toInt) //ismobile
        listMap.put(columnList(4), split(5).trim.replace("\"", "")) //listening_zip_code

        // get customer info from redis
        val cust = r.hmget(split(1).trim, "name", "age", "gender", "zip", "Address", "SignDate", "Status", "Level", "Campaign", "LinkedWithApps")

        // extract detail info and map with elasticsearch field
        listMap.put(columnList(5), cust.get("name"))
        listMap.put(columnList(6), cust.get("age").toInt)
        listMap.put(columnList(7), cust.get("gender"))
        listMap.put(columnList(8), cust.get("zip"))
        listMap.put(columnList(9), cust.get("Address"))
        listMap.put(columnList(10), cust.get("SignDate"))
        listMap.put(columnList(11), cust.get("Status"))
        listMap.put(columnList(12), cust.get("Level"))
        listMap.put(columnList(13), cust.get("Campaign"))
        listMap.put(columnList(14), cust.get("LinkedWithApps"))

        println(s" map = ${listMap.toString()}")
        listMap
      }).iterator
    })

    //[STEP 3]. Write to ElasticSearch
    wordList.foreachRDD(rdd => {
      rdd.foreach(s => s.foreach(x => println(x.toString)))
      EsSpark.saveToEs(rdd, "ba_realtime2/stage2")
    })
    ssc.start()
    ssc.awaitTermination()
  }

  // get current time
  def getTimestamp(): Timestamp = {
    val format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
    new Timestamp(Calendar.getInstance().getTime().getTime)
  }

  // spark stream listener interface
  def addStreamListener(ssc: StreamingContext): Unit = {
    ssc.addStreamingListener(new StreamingListener() {
      override def onBatchSubmitted(batchSubmitted: StreamingListenerBatchSubmitted): Unit = {
        super.onBatchSubmitted(batchSubmitted)
        val batchTime = batchSubmitted.batchInfo.batchTime
        println("[batchSubmitted] " + batchTime.milliseconds)
      }
      override def onBatchStarted(batchStarted: StreamingListenerBatchStarted): Unit = {
        super.onBatchStarted(batchStarted)
      }
      override def onBatchCompleted(batchCompleted: StreamingListenerBatchCompleted): Unit = {
        super.onBatchCompleted(batchCompleted)
      }
    })
  }
}
```


### Compile spark application and run spark streaming
- compile with maven command line
```
> cd ~/demo-spark-analytics/00.stage2/demo-streaming

> sudo yum install -y maven
> mvn package

> ls -alh target
# 필요한 library를 모두 합친 jar 파일이 생성되었다.
-rw-rw-r--. 1 freepsw.12 freepsw.12 110M  4월 11 12:11 demo-streaming-1.0-SNAPSHOT.jar
-rw-rw-r--. 1 freepsw.12 freepsw.12  31K  4월 11 12:10 original-demo-streaming-1.0-SNAPSHOT.jar

* ..-jar-with-dependencies.jar은 application에 필요한 모든 library가 포함된 파일
* spark은 분산 환경에서 구동하기 때문에, 해당 library가 모든 서버에 존재해야만 정상적으로 실행,
* 이러한 문제를 해결하기 위해서 jar파일 내부에 필요한 모든 library를 포함하도록 실행파일 생성.
```

- spark-submit을 통해 spark application을 실행시킨다.
```
> cd ~/demo-spark-analytics/00.stage2
> ./run_spark_streaming_s2.sh
```

- 상세 설정
  - class : jar파일 내부에서 실제 구동할 class명
  - master : spark master의 ip:port(default 7077)
  - deploy-mode : spark는 driver와 executor로 구분되어 동작하게 됨. 여기서 driver의 구동 위치를 결정
  - client : 현재 spark-submit을 실행한 서버에 driver가 구동됨.
  - cluster : spark master가 cluster node 중에서 1개의 node를 지정해서, 해당 node에서 driver 구동
  - driver-memory : driver 프로세스에 할당되는 메모리
  - executor-memory : executor 1개당 할당되는 메모리
  - total-executor-cores : Total cores for all executors.
  - executor-cores : Number of cores per executor
```shell
spark-submit \
  --class io.skiper.driver.Stage2StreamingDriver \
  --master spark://localhost:7077 \
  --name "demo-spark-analysis-s2" \
  --deploy-mode client \
  --driver-memory 1g \
  --executor-memory 1g \
  --total-executor-cores 2 \
  --executor-cores 2 \
  ./demo-streaming/target/demo-streaming-1.0-SNAPSHOT-jar-with-dependencies.jar
```

## [STEP 8] Generate customer log data
### run data_generator
- stage1에서 구동했던 프로세스이므로,
- 만약 현재 구동중이라면 이 단계는 생략한다.  
```
> cd ~/demo-spark-analytics/00.stage1
> python data_generator.py
```


## [STEP 9] visualize collected data using kibana
### kibana 시각화 가이드 참고
- kibana를 이용한 시각화 가이드는 아래의 link에 있는 ppt파일을 참고
- https://github.com/freepsw/demo-spark-analytics/blob/master/00.stage2/kibana_visualization_guide_stage2.pptx
- 이 외에도 다양한 방법으로 시각화가 가능하므로, 실습시간을 이용하여 다양한 실시간 시각화를 진행

### kibana dashbboard하여 시각화
- Kibana 메인메뉴 > Settings > Objects 클릭
- "import"버튼을 클릭하고,
- 00.stage2 폴더 아래에 있는 kibana_dashboard_s2.json 선택
- 기존에 정의된 dashboard를 화면에 시각화하여 보여준다.


## Etc 고려할 시나리오
 * spark : 1 node(client mode), save to ES/redis
  - customerid, trackid와 상세정보를 join(redis)하여 ES에 저장한다.
  - 최근 30시간 동안 남자/여자가 가장 많이 들은 top 10 music
 * customerid, trackid와 상세정보를 join(redis)하여 데이터를 추가한다. -> ES
- compute a summary profile for each user
 * 특정기간(아침, 점심, 저녁)동안 각 사용자들이 들은 음악의 평균값 (언제 가장 많이 듣는가?)
 * 전체 사용자들이 들은 전체 음악 목록 (중복 제거한 unique값)
 * 모바일 기기에서 들은 전체 음악 목록(중복 제거한 unique)
- 특정 시간(30분) 이내에 같은 곡을 3번 이상 들은 사용자는 해당곡을 관심 list로 등록 -> Redis, ES


## [ETC]
### Redis Error 1
- https://league-cat.tistory.com/384
- https://gist.github.com/kapkaev/4619127


#### 현상
- 갑자기 Failed opening .rdb for saving: Permission denied 

#### 원인
- 원래 설정한 redis dir은 default 설정이 redis server를 실행하는 경로임 (dir ./)
- 그런데 일정 시간이 지난후 위의 에러가 발생한 후 redis 설정을 redis-cli에서 확인해 보면, 
- 아래와 같이 /etc로 변경된 것이 보임. (redis.conf 파일에는 ./로 변경사항 없음...)
- 아무도 dir 설정을 바꾸지 않았는데, 갑자기 바뀌다니....??????
```
# 아래 rdb파일을 저장하는 디렉토리를 보면 /etc로 설정되어 있음. 
127.0.0.1:6379> config get dir
1) "dir"
2) "/etc"

# dbfilename은 "dump.rdb"에서 "crontab"으로 변경됨. 
127.0.0.1:6379> config get dbfilename
1) "dbfilename"
2) "crontab"

```

- redis github에 유사한 문제에 대한 답변이 등록됨 (2016년)
- https://github.com/redis/redis/issues/3594
- 핵심 내용은 redis server를 실행하고, 외부에서 접근가능한 네트워크 환경으로 구성하면,
- redis-cli를 통해 누구가 redis-cli 명령을 실행할 수 있음. 
- 따라사 config set dir "/etc"와 같이 원하는 설정으로 변경이 가능함... 헐.... (하긴 redis에 따로 접근제어를 설정한 것이 없으니....)
- 그래서 bot을 이용하여 전 세계 IP와 Port를 탐지하면서 redis를 통해서 공격을 시도하는 경우가 많고,
- 위 redis dir을 바꾸는 것도 이로 인한 결과라고 답변함. 
  - 즉, 쓰기 가능한 디렉토리를 /etc로 변경하고, 
  - 거기에 crontab을 생성하여 원하는 작업(공격)을 하기 위함.  

#### 해결

- 첫번쨰, redis server가 실행되는 서버에 외부에서 접근할 수 없도록 네트워크 제어 (방화벽 등)
- 두전쨰, protected-mode가 적용된 redis V3.2 이상을 사용 
```
bind 127.0.0.1
protected-mode yes
rename-command CONFIG ""

```

### Spark application을  Master Cluster로 실행하기
- SparkConf().setMaster("local[2]") 설정 변경

## local[] 방식 
- 현재 로컬 환경에서 별도로 실행된다.
- 따라서 Spark Cluster UI에서 실행 application이 조회되지 않음. 
- 다만, 실행되는 코드의 모든 로그를 바로 콘솔에서 확인 가능함. 

## 