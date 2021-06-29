---
layout: post
title: 카프카 컨슈머 메트릭 수집
author: Hansuk Hong
author-email: hansuk.hong@cre.ma
excerpt: Burrow + Datadog 으로 Kafka consumers 의 오프셋/랙 등 토픽/파티션 별 메트릭 수집
publish: true
---

안녕하세요. 크리마 DevOps 의 홍한석입니다. 이번 글에서는 크리마가 카프카를 사용하며 카프카 컨슈머 모니터링을 하기 위해 사용한 기술들과 경험을 공유하겠습니다.

현재 크리마는 여러 앱들의 로그 및 로그성 데이터를 카프카에 메세지로 보내 모으고 있습니다.

![CREMA-Kafka-Arch](https://user-images.githubusercontent.com/20619567/123821922-cadeea00-d936-11eb-9b09-3b8d21657441.jpg)

그런데 최근 특정 데이터 수집이 안되는 이슈를 겪으며 카프카 컨슈머 메트릭 모니터링이 필요함을 느꼈습니다. 카프카에 있는 여러 종류의 메세지 중 어떤 종류의 메세지가 잘 처리 안되고 있는지 알기 위해 특정 컨슈머 그룹, 토픽, 파티션의 오프셋과 랙을 지속적으로 수집해보기로 했습니다.

이 글은 다음 내용을 다룹니다:

- Burrow 설치(Ansible tasks/playbook)
- Burrow 사용법
- Datadog Custom Agent Check 제작
- Datadog 제공 Agent Check: Kafka Consumer 설정

## Burrow 설치 및 사용하기

크리마는 Datadog 을 모니터링 도구로 쓰고 있어 여기에 메트릭을 수집할 방법을 찾아보았습니다. "datadog kafka consumer metrics" 라고 구글링 했을 때 발견한 [글](https://www.theteams.kr/teams/865/post/64574)에서 카프카 컨슈머 모니터링 도구인 [Burrow](https://github.com/linkedin/Burrow)에 대해 알게 되었습니다.

링크드인이 개발한 툴이라 믿음이 가기도 했고, Datadog에 메트릭을 생성하는 플러그인도 이미 있어서 사용해보기로 했습니다.

먼저 Burrow 는 Go 로 만들어진 프로그램인데 크리마에서는 Go 를 사용하지 않고 있어서 Go 를 설치했습니다.

크리마는 프로비저닝 도구로 Ansible 을 쓰고 있습니다. [Go 설치 문서](https://golang.org/doc/install)를 참고해 다음과 같이 앤서블 태스크를 만들어 플레이했습니다:

{% raw %}

```yml
---
- name: Check installed Go version
  command: /usr/local/go/bin/go version
  register: check_go_version
  ignore_errors: yes

- name: Install Go
  block:
    - name: Remove the installed old go
      file:
        path: /usr/local/go
        state: absent
      become: true

    - name: Download & Unarchive Go for linux
      unarchive:
        src: https://golang.org/dl/go{{ go_version }}.linux-amd64.tar.gz
        dest: /usr/local
        remote_src: yes
        list_files: yes
        owner: root
        group: root
      become: true
  when:
    - check_go_version is not skipped
    - check_go_version is failed or go_version | string not in check_go_version.stdout

- name: Add go to PATH
  become: yes
  become_user: "{{ item }}"
  blockinfile:
    insertbefore: BOF
    path: "~{{ item }}/.bashrc"
    marker: "# {mark} go ANSIBLE MANAGED BLOCK"
    block: |
      export PATH="$PATH:/usr/local/go/bin"
  with_items: "{{ go_users }}"
```

{% endraw %}

그리고 Burrow 를 설치 및 실행하는 태스크와 설정 파일(burrow.toml)의 템플릿입니다:

{% raw %}

```yml
---
- name: Check Burrow installed
  stat:
    path: "{{ go_bin }}/Burrow"
  register: check_burrow_bin

- name: Install Burrow
  command: /usr/local/go/bin/go install github.com/linkedin/Burrow@v{{ burrow_version }}
  environment:
    GOPATH: "{{ go_path }}"
  when:
    - check_burrow_bin is not skipped
    - not check_burrow_bin.stat.exists

- name: Check Burrow port is using
  wait_for:
    host: localhost
    port: 8000
    state: started
    delay: 0
    timeout: 3
  register: check_burrow_port
  failed_when:
    - check_burrow_port.msg != "Timeout when waiting for localhost:8000"
    - check_burrow_port.state == "started"

- name: Check Burrow is running
  uri:
    url: http://localhost:{{ burrow_port }}/burrow/admin
    return_content: yes
  register: check_burrow_running
  failed_when: check_burrow_running.content == "GOOD"

- name: Create Burrow configuration directory
  file:
    path: "{{ burrow_dir }}"
    state: directory
    mode: 0755

- name: Generate Burrow configuration file
  template:
    src: burrow.toml.j2
    dest: "{{ burrow_dir }}/burrow.toml"
    mode: 0644

- name: Run Burrow
  shell: nohup {{ go_bin }}/Burrow --config-dir {{ burrow_dir }} >> {{ burrow_dir }}/burrow.log 2>&1 &

- name: Create running of Burrow on cron
  cron:
    name: "Burrow for reboot"
    special_time: reboot
    job: "/bin/bash -l -c 'nohup {{ go_bin }}/Burrow --config-dir {{ burrow_dir }} >> {{ burrow_dir }}/burrow.log 2>&1 &'"
```

```toml
[zookeeper]
servers=[ "{{ zookeeper_hosts | join('\", \"') }}" ]
timeout=6
root-path="/burrow"

[client-profile.profile]
kafka-version="0.11.0"
client-id="burrow-client"

[cluster.local]
client-profile="profile"
class-name="kafka"
servers=[ "{{ kafka_hosts | join('\", \"') }}" ]
topic-refresh=60
offset-refresh=30
groups-reaper-refresh=30

[consumer.local]
class-name="kafka"
cluster="local"
servers=[ "{{ kafka_hosts | join('\", \"') }}" ]
group-denylist="^(console-consumer-|python-kafka-consumer-).*$"
group-allowlist=""

[httpserver.default]
address=":{{ burrow_port }}"
```

{% endraw %}

설정 파일은 [docker-config/burrow.toml](https://github.com/linkedin/Burrow/blob/master/docker-config/burrow.toml) 을 바탕으로 카프카와 주키퍼 호스트만 변수로 설정할 수 있게 했습니다.
또 크리마가 쓰는 카프카 2.5.x 버전은 주키퍼를 내부적으로만 쓰고 있어 주키퍼 컨슈머(`class-name=kafka_zk`) 설정은 제거했습니다.
설정 관련한 자세한 내용은 [Burrow Configuration 문서](https://github.com/linkedin/Burrow/wiki/Configuration)를 참조 바랍니다.

이렇게 설치 및 실행을 마치고 나면 Burrow 의 HTTP REST API 로 호출할 수 있습니다. 간단히 health check 와 설정한 클러스터 응답을 확인합니다. health check 를 제외한 모든 응답 형식은 JSON 이라 jq 로 파싱했습니다:

```sh
$ curl -s localhost:8000/burrow/admin
GOOD$
...

$ curl -s localhost:8000/v3/kafka | jq
{
  "error": false,
  "message": "cluster list returned",
  "clusters": [
    "local"
  ],
  "request": {
    "url": "/v3/kafka",
    "host": "<hostaname>"
  }
}
```

더 자세한 [API Endpoint 는 Burrow 문서](https://github.com/linkedin/Burrow/wiki/HTTP-Endpoint)를 확인 바랍니다.

### Burrow Notifier

Burrow 는 notifier 를 내장하고 있어 특정 조건에 알람을 보낼 수 있습니다. 예를 들면 특정 토픽/파티션의 랙이 threshold 이상 올라가면 템플릿 메세지로 슬랙 알람을 보낼 수 있게 설정이 가능합니다.

![Burrow-Slack-Noti](https://user-images.githubusercontent.com/20619567/123822899-a6374200-d937-11eb-9e60-f7685c864296.png)

테스트 해보았지만, 단순한 threshold 알람보다 더 다양하게 활용하기 위해 원래 목표했던 메트릭 수집을 하기로 했습니다. Notifier 설정 관련한 내용은 다음 문서를 참조 바랍니다:

- [Notifier HTTP](https://github.com/linkedin/Burrow/wiki/Notifier-HTTP)
- [Notifier Email](https://github.com/linkedin/Burrow/wiki/Notifier-Email)

## Burrow 를 통해 Datadog Custom Check 를 만들어 카프카 컨슈머 메트릭 수집하기

Burrow API 는 토픽별 그리고 컨슈머별 각 파티션의 오프셋 및 랙 정보를 제공합니다.

[토픽 상세(`GET /v3/kafka/(cluster)/topic/(topic)`)](https://github.com/linkedin/Burrow/wiki/http-request-get-topic-detail) 응답의 `.offsets` 는 각 인덱스가 파티션 번호를 나타내고 값은 오프셋입니다. 아직 오프셋이 없는 파티션은 0으로 채워져 있습니다:

```sh
$ curl -s localhost:8000/v3/kafka/local/topic/topicname | jq .offsets
[
  1171173068,
  898131979,
  700227987,
  664021705,
  839447087,
  843509617
]
```

[컨슈머 상세(`GET /v3/kafka/(cluster)/consumer/(group)`)](https://github.com/linkedin/Burrow/wiki/http-request-get-consumer-detail) 응답의 `.topics` 는 각 토픽 이름의 키와 파티션 수의 배열 값의 JSON 이고, 각 파티션엔 오프셋과 랙 관련한 자세한 정보가 있습니다. 오프셋 배열(`.topics.<topicname>.[*].offsets`)은 마지막 10개를 반환하고 컨슈머가 소비한 전체 오프셋이 10개가 안되면 앞에서부터 null 로 채워져 있습니다:

```sh
$ curl -s localhost:8000/v3/kafka/local/consumer/groupname | jq .topics.topicname[0].offsets
[
  null,
  null,
  null,
  null,
  null,
  null,
  null,
  null,
  {
    "offset": 1171275688,
    "timestamp": 1624975046085,
    "observedAt": 1624975046000,
    "lag": 0
  },
  {
    "offset": 1171276581,
    "timestamp": 1624975051085,
    "observedAt": 1624975051000,
    "lag": 0
  }
]
```

위 API 응답 중 필요한 값을 Datadog Check 스크립트에서 메트릭으로 수집합니다.

이미 [datadog-agent-burrow](https://github.com/packetloop/datadog-agent-burrow) 라는 플러그인이 있어 이를 그대로 가져다 쓰려 했으나 문제가 있었습니다. 이 플러그인은 Burrow API V2 에 대응하는 스크립트라 현재(Burrow v1.3.6) 버전인 API V3 엔 동작하질 않았습니다.

처음엔 Burrow API V2 의 응답을 보고 플러그인 스크립트를 파악하여 차이점을 발견하고 수정하고자 했으나 API V2 는 Burrow 가 정식 1.0.0 버전 릴리즈 하기 전에 쓰던 인터페이스였고, 그전 커밋이 잘게 나뉘어 있지 않아 추적하는게 까다로웠습니다. 따라서 기존 플러그인 스크립트를 파악하고 V3 에 대응하도록 고쳐 쓰게 되었습니다.

[Burrow API V3 에 맞춰 수정한 스크립트](https://github.com/flavono123/datadog-agent-burrow/blob/feature/upgrade-comp-to-vthree/checks.d/burrow.py)입니다. [PR](https://github.com/packetloop/datadog-agent-burrow/pull/14) 도 열어두었으나 아마 확인도 안하거나 merge 되긴 어려울거 같습니다(과거에 열린 비슷한 내용의 PR 이 있는데, 댓글로 [회사 내부에선 Burrow V2를 쓰고 있다](https://github.com/packetloop/datadog-agent-burrow/pull/9#discussion_r299752102)고 하네요 ^^..)

이제 Datadog Custom Agent Check 로 설치하고 테스트해봅니다. 크리마는 Agent V5 를 쓰고 있어서 다음과 같이 테스트했습니다:

```sh
$ sudo -u dd-agent -- dd-agent check burrow
...
Metrics:
[('kafka.topic.offsets',
  1624976795,
  0,
  {'hostname': '<hostname>',
   'tags': ('cluster:local', 'partition:0', u'topic:<topicname>'),
   'type': 'gauge'}),
 ('kafka.topic.offsets.total',
  1624976795,
  0,
  {'hostname': '<hostname>',
   'tags': ('cluster:local', u'topic:<topicname>'),
   'type': 'gauge'})]
Events:
[]
Service Checks:
[{'check': 'burrow.can_connect',
  'host_name': '<hostname>',
  'id': 1,
  'message': u'Connection to http://localhost:8000/burrow/admin was successful',
  'status': 0,
  'tags': ['instance:<hostname>'],
  'timestamp': 1624976795.910263}]
Service Metadata:
[{}]
    burrow (5.32.7)
    ---------------
      - instance #0 [OK]
      - Collected 2 metrics, 0 events & 1 service check
```

## Datadog 제공 Agent Check: Kafka Consumer 를 통해 메트릭 수집하기

Burrow 를 통해 Custom Agent Check 로 수집하지 않아도 [Datadog 이 제공하는 Agent Check](https://docs.datadoghq.com/integrations/kafka/?tab=host#agent-check-kafka-consumer) 가 있어 설정으로 간단히 오프셋과 랙을 수집할 수 있습니다.

[설정 문서](https://github.com/DataDog/integrations-core/blob/master/kafka_consumer/datadog_checks/kafka_consumer/data/conf.yaml.example)를 참고해 추가했는데 이때 **주의할 점은 `zk_connect_str` 을 설정하지 않아야 컨슈머 메트릭이 잘 수집됩니다.** `zk_connect_str` 을 설정하면 `kafka.broker_offset` 만 수집되고 `kafka.consumer_offset/lag` 은 수집이 안되는데요. 이는 카프카가 계속 주키퍼 의존성을 제거하며 더 이상 컨슈머 오프셋을 주키퍼에 저장하지 않기 때문이라고 문서 deprecation 에서 잘 설명하고 있습니다.

또 `consumer_groups` param 에서 empty(`{}`) mapping 을 주어 모든 토픽/파티션을 수집하는게 잘 동작하지 않아 토픽을 일일이 써 주었습니다:

```yml
# Ansible managed
init_config: null
instances:
-   consumer_groups:
        <grouname1>:
            <topicname1>: []
        <groupname2>:
            <topciname1>: []
            ...
    kafka_connect_str:
    - <kafkahostname1>:9092
    - <kafkahostname1>:9092
    - <kafkahostname1>:9092
```

수집한 메트릭 중 `kafka.consumer_lag` 을 다음처럼 각각 Timeseries 대시보드에 모니터하기 시작했습니다(컨슈머 그룹, 토픽, 파티션은 각각 태그가 생깁니다):

1. 한 토픽에 여러 파티션이 나뉜 컨슈머 그룹은 파티션별로:
   ![TS-avg-partition](https://user-images.githubusercontent.com/20619567/123823155-e4346600-d937-11eb-978e-7abc60fb3f6c.png)

2. 컨슈머 그룹이 여러 토픽을 소비하는 경우 토픽별로:
   ![TS-avg-topic](https://user-images.githubusercontent.com/20619567/123823214-f3b3af00-d937-11eb-8eb8-517ae4a14a88.png)

이제 랙을 모니터를 하기 시작했으니 어느 시간대에 어떤 토픽/파티션의 처리가 느린지 파악하기 쉬워졌습니다. 패턴을 지켜보고 알람을 추가하거나 병목 지점을 개선해 나가는데 도움이 될거 같습니다.

## 후기

Burrow 를 사용해 Custom Agent Check 를 연동해 가던 중 Datadog 이 Agent Check 를 제공한다는 사실을 알게 되었습니다(삽질...). 하지만 이전엔 파이썬 커스텀 체크 스크립트를 만들어 본적이 없기도 하고, 기반이 되는 낡은 플러그인이 있어 좋은 기회라는 생각도 들었습니다. 한번 커스텀 체크 제작을 경험해보자라는 마음으로 끝까지 하게 되었네요. 또 Burrow API 는 오프셋과 랙 뿐만 아니라 랙 상태(UNKNOWN, OK, WARN, ... ) 같은 추가적인 정보도 제공하고 단독적으로도 좋은 툴이라 그렇게 헛수고는 아니었던것 같습니다.

또 블로그 글을 통해 코드를 외부에 공개된다고 의식을 하니 좀 더 코드를 잘 다듬게 되었습니다. 내부에서도 코드 리뷰를 하지만, 특히 Ansible 변수를 추출하는 과정이 더 일반화 되어 모듈화가 잘 된거 같습니다.

이 글이 카프카 컨슈머 메트릭을 모니터링하는데 도움이 됐으면 합니다. 긴 글 읽어주셔서 고맙습니다.

### 링크가 빠진 참고

- [Collecting Kafka performance metrics](https://www.datadoghq.com/blog/collecting-kafka-performance-metrics/#monitor-consumer-health-with-burrow)
- [Custom Agent Check](https://docs.datadoghq.com/developers/write_agent_check/?tab=agentv5)
