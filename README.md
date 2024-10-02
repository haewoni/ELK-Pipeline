

# <p align="center"> ELK-Pipeline

### ELK Stack을 활용하여 데이터 시각화하기

## 팀원 🧡 

*신혜원, 이정민, 최나영, 허예은*

## 사용기술
![Spring](https://img.shields.io/badge/Spring-6DB33F?style=for-the-badge&logo=spring&logoColor=white) ![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![k8s](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white) ![]()

<br>



## 프로젝트 목적 🌷
<br>

<br>

## 실습 개요 :star:

- step 01 : 
- step 02 : 
<br>

## 실습 과정 :mag_right:

## 0. 준비

### 설치 가능한 패키지 버전 확인

```bash
apt list -a (패키지명)
```

## 1. Elasticsearch

> https://www.elastic.co/guide/en/elasticsearch/reference/7.11/targz.html
> 

### Download and install archive for Linux

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.2-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.2-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.11.2-linux-x86_64.tar.gz.sha512 
tar -xzf elasticsearch-7.11.2-linux-x86_64.tar.gz
cd elasticsearch-7.11.2/ 
```

![image](https://github.com/user-attachments/assets/216344dd-9d75-4582-ab29-2c468b2f14f1)


![image](https://github.com/user-attachments/assets/641b0d40-f289-4980-8072-280963548b79)


### Run

```bash
./bin/elasticsearch
```

### Test - GET

```bash
curl -X GET "localhost:9200/?pretty"
```

![image](https://github.com/user-attachments/assets/f9c85bdf-fae4-42f8-9538-eb720a5fc1a5)


![image](https://github.com/user-attachments/assets/0f143853-21ca-4be4-bab3-b544bc2b3db1)


## 2. Kibana

> https://www.elastic.co/guide/en/kibana/7.11/install.html
> 

### Download and install the Linux 64-bit package

```bash
curl -O https://artifacts.elastic.co/downloads/kibana/kibana-7.11.2-linux-x86_64.tar.gz
curl https://artifacts.elastic.co/downloads/kibana/kibana-7.11.2-linux-x86_64.tar.gz.sha512 | shasum -a 512 -c - 
tar -xzf kibana-7.11.2-linux-x86_64.tar.gz
cd kibana-7.11.2-linux-x86_64/ 
```

![image](https://github.com/user-attachments/assets/be07a221-bd7d-497f-99a9-8d9b6756f3d5)


### Run

```bash
./bin/kibana
```

![image](https://github.com/user-attachments/assets/a514b5ed-509e-46fd-9d0d-6d3c03bd22a8)


## 3. Logstash

> https://www.elastic.co/guide/en/logstash/7.11/installing-logstash.html
> 

### Install APT

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update && sudo apt-get install logstash=7.11.2
```

### Run

```bash
sudo systemctl enable logstash
sudo systemctl start logstash
```

### Test

```bash
sudo systemctl status logstash
```

![image](https://github.com/user-attachments/assets/156d928b-d2d9-4293-868a-a8d79415dacf)


### Setting

```bash
sudo vi /etc/logstash/conf.d/bank.conf
```

```bash
input {
 # beat에서 데이터를 받을 port지정
  beats {
    port => 5044 
  }
}

filter {
  mutate {
    # 실제 데이터는 "message" 필드로 오기 때문에 csv형태의 내용을 분할하여 새로운 이름으로 필드를 추가 
    # 20180601,NH농협은행,1호점,종각,2314
    split => [ "message",  "," ] 
    add_field => {
      "date" => "%{[message][0]}"
      "bank" => "%{[message][1]}"
      "branch" => "%{[message][2]}"
      "location" => "%{[message][3]}"
      "customers" => "%{[message][4]}"
    }

    # 기본으로 전송되는 데이터 분석에 불필요한 필드는 제거한다. "message" 필드도 위에서 재 가공 했으니 제거
    # head에서 보고 불필요한 속성들 삭제 리스트에 저장해서 관리
    remove_field => ["ecs", "host", "@version", "agent", "log", "tags",  "input", "message"]
  }

  # "date" 필드를 이용하여 Elasticsearch에서 인식할 수 있는 date 타입의 형태로 필드를 추가
  date {
    match => [ "date", "yyyyMMdd"]
    timezone => "Asia/Seoul"
    locale => "ko"
    target => "date"
  }

  # Kibana에서 데이터 분석시 필요하기 때문에 숫자 타입으로 변경
  mutate {
    convert => {
      "customers" => "integer"
    }
    remove_field => [ "@timestamp" ]
  }
}

output {

  # 콘솔창에 어떤 데이터들로 필터링 되었는지 확인
  stdout {
    codec => rubydebug
  }

  # 위에서 설치한 Elasticsearch 로 "bank" 라는 이름으로 인덱싱 
  elasticsearch {
    hosts => ["http://localhost:9200"]
    **index => "bank"**
  }
}
```

### Setting Test

```bash
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

오류가 없다면 `Using config.test_and_exit mode. Config Validation Result: OK. Exiting Logstash` 가 출력된다.

![image](https://github.com/user-attachments/assets/f4fb569b-655d-4c26-9ee1-7e4139f07eb6)

## 4. Beats

> https://www.elastic.co/guide/en/beats/filebeat/7.11/filebeat-installation-configuration.html
> 

### Install

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.11.2-linux-x86_64.tar.gz
tar xzvf filebeat-7.11.2-linux-x86_64.tar.gz
```

![image](https://github.com/user-attachments/assets/9a1766f0-1944-4191-939c-7dfc446b323f)


or

```bash
sudo apt-get update && sudo apt-get install filebeat=7.11.2
```

### Run

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

### Test

```bash
sudo systemctl status filebeat
```

![image](https://github.com/user-attachments/assets/832ec758-f6c8-4b65-9b4b-99d34f5d05a5)

### Setting

```bash
sudo vi /etc/filebeat/filebeat.yml
```

![image](https://github.com/user-attachments/assets/cd536393-903a-47d7-9987-0f0d588b5b32)


주석 해제

## Beats → Logstash → ES

### csv 파일 생성

![image](https://github.com/user-attachments/assets/844d2c02-f611-4233-926b-943390d8455c)


### filebeat.yml 수정

![image](https://github.com/user-attachments/assets/a897263d-055f-4849-8c23-ee0acb4460bd)


![image](https://github.com/user-attachments/assets/c49e90c7-ae6d-4359-a1f0-33aa3a5c9e9c)


### logstash 실행

```bash
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/bank.conf --path.settings /etc/logstash
```

![image](https://github.com/user-attachments/assets/03a736ba-d329-456f-bbb1-61ac5f9d7260)

![image](https://github.com/user-attachments/assets/0d7e68de-36f1-489c-80bb-122b039376bf)


---

## 🧨 트러블슈팅

- ES, Kibana 실행 시 yml을 수정하지 않아도 정상적으로 수행되는 사람과 yml을 수정해야 실행되는 사람이 있었음
    
    → 원래는 포트포워딩 필요
    
    (설정하지 않은 경우에는 자동으로 열렸을 수도 있음)
    
- 원인을 제대로 알 수 없는 에러가 발생 → 에러 로그 확인하여 해결
    
    ```bash
    sudo journalctl -u elasticsearch
    ```
    
- logstash 실행 오류 → 버전 불일치 (8.xx)

---

## 📚 References

> ELK 7.11 버전 설치
> 
> 
> https://www.elastic.co/guide/en/elastic-stack/7.11/installing-elastic-stack.html
>

<br>
