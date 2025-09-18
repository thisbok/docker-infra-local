# Docker 기반 Elastic Stack (ELK) + Redis + MySQL

[![Elastic Stack version](https://img.shields.io/badge/Elastic%20Stack-8.10.4-00bfb3?style=flat&logo=elastic-stack)](https://www.elastic.co/blog/category/releases)
[![Build Status](https://github.com/deviantony/docker-elk/workflows/CI/badge.svg?branch=main)](https://github.com/deviantony/docker-elk/actions?query=workflow%3ACI+branch%3Amain)
[![Join the chat](https://badges.gitter.im/Join%20Chat.svg)](https://app.gitter.im/#/room/#deviantony_docker-elk:gitter.im)

Docker 와 Docker Compose 를 사용하여 최신 버전의 [Elastic stack][elk-stack]을 Redis 와 MySQL 과 함께 실행합니다.

Elasticsearch 의 검색/집계 기능과 Kibana 의 시각화 기능을 사용하여 모든 데이터셋을 분석할 수 있습니다.

Elastic 의 [공식 Docker 이미지][elastic-docker]를 기반으로 합니다:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/main/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/main/docker)
* [Kibana](https://github.com/elastic/kibana/tree/main/src/dev/build/tasks/os_packages/docker_generator)

기타 사용 가능한 스택 변형:

* [`tls`](https://github.com/deviantony/docker-elk/tree/tls): Elasticsearch, Kibana(선택 사항), Fleet 에서 TLS 암호화 활성화
* [`searchguard`](https://github.com/deviantony/docker-elk/tree/searchguard): Search Guard 지원

> [!IMPORTANT]
> [Platinum][subscriptions] 기능은 기본적으로 **30 일** [평가판][license-mngmt] 기간 동안 활성화됩니다. 평가 기간 이후에는 수동 개입 없이 자동으로 Open Basic 라이선스에 포함된 모든 무료 기능에 액세스할 수 있으며, 데이터 손실 없이 원활하게 전환됩니다. 이 동작을 비활성화하려면 [유료 기능 비활성화 방법](#how-to-disable-paid-features) 섹션을 참조하세요.

---

## 빠른 시작

```sh
docker-compose up setup
```

```sh
docker-compose up
```

![Animated demo](https://user-images.githubusercontent.com/3299086/155972072-0c89d6db-707a-47a1-818b-5f976565f95a.gif)

---

## 철학

이 강력한 기술 조합을 실험해보고 싶은 모든 사람에게 Elastic stack 에 대한 가장 간단한 진입점을 제공하는 것을 목표로 합니다. 이 프로젝트의 기본 구성은 의도적으로 최소한이고 고정되지 않았습니다. 외부 종속성에 의존하지 않으며, 서비스를 시작하고 실행하는 데 필요한 최소한의 사용자 정의 자동화를 사용합니다.

대신, 이 저장소를 템플릿으로 사용하여 수정하고 _자신만의_ 것으로 만들 수 있도록 좋은 문서를 제공한다고 믿습니다. [sherifabdlnaby/elastdocker][elastdocker]는 이러한 아이디어를 기반으로 구축된 프로젝트의 한 예입니다.

---

## 목차

1. [요구사항](#요구사항)
   * [호스트 설정](#호스트-설정)
   * [Docker Desktop](#docker-desktop)
     * [Windows](#windows)
     * [macOS](#macos)
1. [사용법](#사용법)
   * [스택 시작하기](#스택-시작하기)
   * [초기 설정](#초기-설정)
     * [사용자 인증 설정](#사용자-인증-설정)
     * [데이터 주입](#데이터-주입)
   * [정리하기](#정리하기)
   * [버전 선택](#버전-선택)
1. [구성](#구성)
   * [Elasticsearch 구성하는 방법](#elasticsearch-구성하는-방법)
   * [Kibana 구성하는 방법](#kibana-구성하는-방법)
   * [Logstash 구성하는 방법](#logstash-구성하는-방법)
   * [유료 기능 비활성화하는 방법](#유료-기능-비활성화하는-방법)
   * [Elasticsearch 클러스터 확장하는 방법](#elasticsearch-클러스터-확장하는-방법)
   * [설정 재실행하는 방법](#설정-재실행하는-방법)
   * [프로그래밍 방식으로 비밀번호 재설정하는 방법](#프로그래밍-방식으로-비밀번호-재설정하는-방법)
1. [확장성](#확장성)
   * [플러그인 추가하는 방법](#플러그인-추가하는-방법)
   * [제공된 확장 기능 활성화하는 방법](#제공된-확장-기능-활성화하는-방법)
1. [JVM 튜닝](#jvm-튜닝)
   * [서비스가 사용할 메모리 양 지정하는 방법](#서비스가-사용할-메모리-양-지정하는-방법)
   * [서비스에 원격 JMX 연결 활성화하는 방법](#서비스에-원격-jmx-연결-활성화하는-방법)
1. [더 나아가기](#더-나아가기)
   * [플러그인 및 통합](#플러그인-및-통합)
1. [Redis 및 MySQL 통합](#redis-및-mysql-통합)
   * [아키텍처 개요](#아키텍처-개요)
   * [데이터 흐름](#데이터-흐름)
   * [MySQL 설정](#mysql-설정)

## 요구사항

### 호스트 설정

* [Docker Engine][docker-install] 버전 **18.06.0** 이상
* [Docker Compose][compose-install] 버전 **1.28.0** 이상 ([Compose V2][compose-v2] 포함)
* 1.5 GB RAM

> [!NOTE]
> 특히 Linux 에서는 사용자가 Docker 데몬과 상호 작용할 수 있는 [필요한 권한][linux-postinstall]을 가지고 있는지 확인하세요.

기본적으로 스택은 다음 포트를 노출합니다:

* 5044: Logstash Beats 입력
* 50000: Logstash TCP 입력
* 9600: Logstash 모니터링 API
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP 전송
* 5601: Kibana
* 6379: Redis
* 3306: MySQL

> [!WARNING]
> 개발 환경에서 Elastic stack 설정을 용이하게 하기 위해 Elasticsearch 의 [부트스트랩 검사][bootstrap-checks]가 의도적으로 비활성화되었습니다. 프로덕션 설정의 경우, Elasticsearch 문서의 지침에 따라 호스트를 설정하는 것을 권장합니다: [중요한 시스템 구성][es-sys-config].

### Docker Desktop

#### Windows

_Docker Desktop for Windows_의 레거시 Hyper-V 모드를 사용하는 경우, `C:` 드라이브에 대해 [파일 공유][win-filesharing]가 활성화되어 있는지 확인하세요.

#### macOS

_Docker Desktop for Mac_의 기본 구성은 `/Users/`, `/Volume/`, `/private/`, `/tmp` 및 `/var/folders`에서만 파일 마운트를 허용합니다. 저장소가 이러한 위치 중 하나에 복제되었는지 확인하거나 [문서][mac-filesharing]의 지침에 따라 더 많은 위치를 추가하세요.

## 사용법

> [!WARNING]
> 브랜치를 전환하거나 기존 스택의 [버전](#버전-선택) 을 업데이트할 때마다 `docker-compose build`로 스택 이미지를 다시 빌드해야 합니다.

### 스택 시작하기

아래 명령으로 스택을 실행할 Docker 호스트에 이 저장소를 복제합니다:

```sh
git clone https://github.com/deviantony/docker-elk.git
```

그런 다음 docker-elk 에 필요한 Elasticsearch 사용자 및 그룹을 초기화하는 명령을 실행합니다:

```sh
docker-compose up setup
```

모든 것이 잘 진행되고 설정이 오류 없이 완료되면, 다른 스택 구성 요소를 시작합니다:

```sh
docker-compose up
```

> [!NOTE]
> 위 명령에 `-d` 플래그를 추가하여 모든 서비스를 백그라운드 (분리 모드) 에서 실행할 수도 있습니다.

Kibana 가 초기화될 때까지 약 1 분 정도 기다린 다음, 웹 브라우저에서 <http://localhost:5601>을 열어 Kibana 웹 UI 에 액세스하고 다음 (기본) 자격 증명을 사용하여 로그인하세요:

* 사용자: _elastic_
* 비밀번호: _changeme_

> [!NOTE]
> 초기 시작 시 `elastic`, `logstash_internal` 및 `kibana_system` Elasticsearch 사용자는 [`.env`](.env) 파일에 정의된 비밀번호 값으로 초기화됩니다 (기본값은 _"changeme"_). 첫 번째는 [내장 슈퍼유저][builtin-users]이고, 나머지 두 개는 각각 Kibana 와 Logstash 가 Elasticsearch 와 통신하는 데 사용됩니다. 이 작업은 스택의 _초기_ 시작 중에만 수행됩니다. 초기화 _후_ 사용자 비밀번호를 변경하려면 다음 섹션의 지침을 참조하세요.

### 초기 설정

#### 사용자 인증 설정

> [!NOTE]
> 인증을 비활성화하려면 [Elasticsearch 의 보안 설정][es-security]을 참조하세요.

> [!WARNING]
> Elastic v8.0.0 부터는 부트스트랩된 권한이 있는 `elastic` 사용자를 사용하여 Kibana 를 실행할 수 없습니다.

앞서 언급한 모든 사용자에 대해 기본적으로 설정된 _"changeme"_ 비밀번호는 **안전하지 않습니다**. 보안을 강화하기 위해 앞서 언급한 모든 Elasticsearch 사용자의 비밀번호를 임의의 비밀번호로 재설정할 것입니다.

1. 기본 사용자 비밀번호 재설정

    아래 명령은 `elastic`, `logstash_internal` 및 `kibana_system` 사용자의 비밀번호를 재설정합니다. 이를 기록해 두세요.

    ```sh
    docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic
    ```

    ```sh
    docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user logstash_internal
    ```

    ```sh
    docker-compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system
    ```

    필요한 경우 (예: Beats 및 기타 구성 요소를 통해 [모니터링 정보를 수집][ls-monitoring]하려는 경우) 나머지 [내장 사용자][builtin-users]에 대해서도 언제든지 이 작업을 반복할 수 있습니다.

1. 구성 파일에서 사용자 이름 및 비밀번호 교체

    `.env` 파일 내의 `elastic` 사용자 비밀번호를 이전 단계에서 생성된 비밀번호로 교체하세요. 이 값은 핵심 구성 요소에서 사용되지 않지만 [확장 기능](#제공된-확장-기능-활성화하는-방법) 에서 Elasticsearch 에 연결하는 데 사용됩니다.

    > [!NOTE]
    > 제공된 [확장 기능](#제공된-확장-기능-활성화하는-방법) 을 사용할 계획이 없거나 이러한 서비스를 인증하기 위해 자체 역할과 사용자를 생성하는 것을 선호하는 경우, 스택이 초기화된 후 `.env` 파일에서 `ELASTIC_PASSWORD` 항목을 완전히 제거하는 것이 안전합니다.

    `.env` 파일 내의 `logstash_internal` 사용자 비밀번호를 이전 단계에서 생성된 비밀번호로 교체하세요. 이 값은 Logstash 파이프라인 파일 (`logstash/pipeline/logstash.conf`) 내에서 참조됩니다.

    `.env` 파일 내의 `kibana_system` 사용자 비밀번호를 이전 단계에서 생성된 비밀번호로 교체하세요. 이 값은 Kibana 구성 파일 (`kibana/config/kibana.yml`) 내에서 참조됩니다.

    이러한 구성 파일에 대한 자세한 정보는 아래 [구성](#구성) 섹션을 참조하세요.

1. Logstash 와 Kibana 를 다시 시작하여 새 비밀번호를 사용하여 Elasticsearch 에 다시 연결

    ```sh
    docker-compose up -d logstash kibana
    ```

> [!NOTE]
> Elastic stack 의 보안에 대해 자세히 알아보려면 [Elastic Stack 보안][sec-cluster]을 참조하세요.

#### 데이터 주입

웹 브라우저에서 <http://localhost:5601>을 열어 Kibana 웹 UI 를 시작하고 다음 자격 증명을 사용하여 로그인하세요:

* 사용자: _elastic_
* 비밀번호: _\<생성된 elastic 비밀번호>_

이제 스택이 완전히 구성되었으므로 로그 항목을 주입할 수 있습니다.

제공된 Logstash 구성을 사용하면 TCP 포트 50000 을 통해 데이터를 전송할 수 있습니다. 예를 들어, 설치된 `nc` (Netcat) 버전에 따라 다음 명령 중 하나를 사용하여 Logstash 를 통해 Elasticsearch 에 로그 파일 `/path/to/logfile.log`의 내용을 수집할 수 있습니다:

```sh
# nc 버전을 확인하려면 `nc -h`를 실행하세요

cat /path/to/logfile.log | nc -q0 localhost 50000          # BSD
cat /path/to/logfile.log | nc -c localhost 50000           # GNU
cat /path/to/logfile.log | nc --send-only localhost 50000  # nmap
```

Kibana 설치에서 제공하는 샘플 데이터를 로드할 수도 있습니다.

### 정리하기

Elasticsearch 데이터는 기본적으로 볼륨 내부에 유지됩니다.

스택을 완전히 종료하고 모든 지속 데이터를 제거하려면 다음 Docker Compose 명령을 사용하세요:

```sh
docker-compose down -v
```

### 버전 선택

이 저장소는 Elastic stack 의 최신 버전과 일치합니다. `main` 브랜치는 현재 주요 버전 (8.x) 을 추적합니다.

핵심 Elastic 구성 요소의 다른 버전을 사용하려면 [`.env`](.env) 파일 내의 버전 번호를 변경하기만 하면 됩니다. 기존 스택을 업그레이드하는 경우 `docker-compose build` 명령을 사용하여 모든 컨테이너 이미지를 다시 빌드하는 것을 잊지 마세요.

> [!IMPORTANT]
> 스택 업그레이드를 수행하기 전에 각 개별 구성 요소에 대한 [공식 업그레이드 지침][upgrade]에 항상 주의하세요.

이전 주요 버전도 별도의 브랜치에서 지원됩니다:

* [`release-7.x`](https://github.com/deviantony/docker-elk/tree/release-7.x): 7.x 시리즈
* [`release-6.x`](https://github.com/deviantony/docker-elk/tree/release-6.x): 6.x 시리즈 (수명 종료)
* [`release-5.x`](https://github.com/deviantony/docker-elk/tree/release-5.x): 5.x 시리즈 (수명 종료)

## 구성

> [!IMPORTANT]
> 구성은 동적으로 다시 로드되지 않으므로 구성 변경 후 개별 구성 요소를 다시 시작해야 합니다.

### Elasticsearch 구성하는 방법

Elasticsearch 구성은 [`elasticsearch/config/elasticsearch.yml`][config-es]에 저장됩니다.

Compose 파일 내에서 환경 변수를 설정하여 재정의하려는 옵션을 지정할 수도 있습니다:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Docker 컨테이너 내에서 Elasticsearch 를 구성하는 방법에 대한 자세한 내용은 다음 문서 페이지를 참조하세요: [Docker 로 Elasticsearch 설치][es-docker].

### Kibana 구성하는 방법

Kibana 기본 구성은 [`kibana/config/kibana.yml`][config-kbn]에 저장됩니다.

Compose 파일 내에서 환경 변수를 설정하여 재정의하려는 옵션을 지정할 수도 있습니다:

```yml
kibana:

  environment:
    SERVER_NAME: kibana.example.org
```

Docker 컨테이너 내에서 Kibana 를 구성하는 방법에 대한 자세한 내용은 다음 문서 페이지를 참조하세요: [Docker 로 Kibana 설치][kbn-docker].

### Logstash 구성하는 방법

Logstash 구성은 [`logstash/config/logstash.yml`][config-ls]에 저장됩니다.

Compose 파일 내에서 환경 변수를 설정하여 재정의하려는 옵션을 지정할 수도 있습니다:

```yml
logstash:

  environment:
    LOG_LEVEL: debug
```

Docker 컨테이너 내에서 Logstash 를 구성하는 방법에 대한 자세한 내용은 다음 문서 페이지를 참조하세요: [Docker 용 Logstash 구성][ls-docker].

### 유료 기능 비활성화하는 방법

만료일 이전에 진행 중인 평가판을 취소하고 기본 라이선스로 되돌릴 수 있습니다. Kibana 의 [라이선스 관리][license-mngmt] 패널에서 하거나 Elasticsearch 의 `start_basic` [라이선스 API][license-apis]를 사용하세요. 두 번째 옵션은 평가판 만료일 이전에 라이선스가 `basic`으로 전환되거나 업그레이드되지 않으면 Kibana 에 대한 액세스를 복구하는 유일한 방법입니다.

Elasticsearch 의 `xpack.license.self_generated.type` 설정 값을 `trial`에서 `basic`으로 전환하여 라이선스 유형을 변경하는 것 ([라이선스 설정][license-settings] 참조) 은 **초기 설정 이전에 수행된 경우에만** 작동합니다. 평가판이 시작된 후에는 `trial`에서 `basic`으로의 기능 손실을 첫 번째 단락에서 설명한 두 가지 방법 중 하나를 사용하여 _반드시_ 인정해야 합니다.

### Elasticsearch 클러스터 확장하는 방법

Wiki 의 지침을 따르세요: [Elasticsearch 확장](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

### 설정 재실행하는 방법

설정 컨테이너를 다시 실행하고 `.env` 파일에 비밀번호가 정의된 모든 사용자를 다시 초기화하려면 `setup` Compose 서비스를 다시 "up"하면 됩니다:

```console
$ docker-compose up setup
 ⠿ Container docker-elk-elasticsearch-1  Running
 ⠿ Container docker-elk-setup-1          Created
Attaching to docker-elk-setup-1
...
docker-elk-setup-1  | [+] User 'monitoring_internal'
docker-elk-setup-1  |    ⠿ User does not exist, creating
docker-elk-setup-1  | [+] User 'beats_system'
docker-elk-setup-1  |    ⠿ User exists, setting password
docker-elk-setup-1 exited with code 0
```

### 프로그래밍 방식으로 비밀번호 재설정하는 방법

어떤 이유로든 Kibana 를 사용하여 사용자 ([내장 사용자][builtin-users] 포함) 의 비밀번호를 변경할 수 없는 경우, 대신 Elasticsearch API 를 사용하여 동일한 결과를 얻을 수 있습니다.

아래 예에서는 `elastic` 사용자의 비밀번호를 재설정합니다 (URL 의 "/user/elastic"에 주목):

```sh
curl -XPOST -D- 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<현재 elastic 비밀번호> \
    -d '{"password" : "<새 비밀번호>"}'
```

## 확장성

### 플러그인 추가하는 방법

ELK 구성 요소에 플러그인을 추가하려면:

1. 해당 `Dockerfile`에 `RUN` 문을 추가합니다 (예: `RUN logstash-plugin install logstash-filter-json`)
1. 서비스 구성에 관련 플러그인 코드 구성을 추가합니다 (예: Logstash 입력/출력)
1. `docker-compose build` 명령을 사용하여 이미지를 다시 빌드합니다

### 제공된 확장 기능 활성화하는 방법

[`extensions`](extensions) 디렉토리 내에 몇 가지 확장 기능이 있습니다. 이러한 확장 기능은 표준 Elastic stack 의 일부가 아니지만 추가 통합으로 풍부하게 만드는 데 사용할 수 있는 기능을 제공합니다.

이러한 확장 기능에 대한 문서는 각 개별 하위 디렉토리 내에 확장별로 제공됩니다. 일부는 기본 ELK 구성에 대한 수동 변경이 필요합니다.

## JVM 튜닝

### 서비스가 사용할 메모리 양 지정하는 방법

Elasticsearch 와 Logstash 의 시작 스크립트는 환경 변수 값에서 추가 JVM 옵션을 추가할 수 있어 사용자가 각 구성 요소에서 사용할 수 있는 메모리 양을 조정할 수 있습니다:

| 서비스        | 환경 변수     |
|---------------|---------------|
| Elasticsearch | ES_JAVA_OPTS  |
| Logstash      | LS_JAVA_OPTS  |

메모리가 부족한 환경 (Docker Desktop for Mac 은 기본적으로 2GB 만 사용 가능) 을 수용하기 위해 `docker-compose.yml` 파일에서 힙 크기 할당이 기본적으로 Elasticsearch 는 512MB, Logstash 는 256MB 로 제한됩니다. 기본 JVM 구성을 재정의하려면 `docker-compose.yml` 파일에서 일치하는 환경 변수를 편집하세요.

예를 들어, Logstash 의 최대 JVM 힙 크기를 늘리려면:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xms1g -Xmx1g
```

이러한 옵션이 설정되지 않은 경우:

* Elasticsearch 는 [자동으로 결정되는][es-heap] JVM 힙 크기로 시작합니다.
* Logstash 는 고정된 1GB JVM 힙 크기로 시작합니다.

### 서비스에 원격 JMX 연결 활성화하는 방법

Java 힙 메모리와 마찬가지로 (위 참조) JVM 옵션을 지정하여 JMX 를 활성화하고 Docker 호스트에 JMX 포트를 매핑할 수 있습니다.

다음 내용으로 `{ES,LS}_JAVA_OPTS` 환경 변수를 업데이트하세요 (JMX 서비스를 18080 포트에 매핑했으며, 이를 변경할 수 있습니다). Docker 호스트의 IP 주소로 `-Djava.rmi.server.hostname` 옵션을 업데이트하는 것을 잊지 마세요 (**DOCKER_HOST_IP** 교체):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## 더 나아가기

### 플러그인 및 통합

다음 Wiki 페이지를 참조하세요:

* [외부 애플리케이션](https://github.com/deviantony/docker-elk/wiki/External-applications)
* [인기 있는 통합](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)

## Redis 및 MySQL 통합

### 아키텍처 개요

이 프로젝트는 표준 ELK 스택을 Redis 와 MySQL 로 확장합니다:

* **Redis**: 메시지 큐 역할을 하며 JSON 형식의 데이터를 리스트에 저장
* **MySQL**: 애플리케이션 데이터를 위한 관계형 데이터베이스
* **Logstash**: Redis 에서 데이터를 가져와 처리한 후 Elasticsearch 로 전송

### 데이터 흐름

1. 애플리케이션이 JSON 데이터를 Redis 의 `transactions` 키에 저장
2. Logstash 가 Redis 에서 데이터를 주기적으로 가져옴
3. 데이터가 Logstash 필터를 통해 처리됨
4. 처리된 데이터가 Elasticsearch 로 전송됨
5. Kibana 에서 데이터 시각화 및 분석

### MySQL 설정

MySQL 은 다음과 같이 구성됩니다:

* **데이터베이스**: `elk_db`
* **사용자**: `elk_user`
* **포트**: 3306
* **자격 증명**: `.env` 파일에서 구성

MySQL 에 연결하려면:

```sh
docker-compose exec mysql mysql -u elk_user -p elk_db
```

Redis 에 연결하려면:

```sh
docker-compose exec redis redis-cli
```

Redis 에 테스트 데이터를 추가하려면:

```sh
docker-compose exec redis redis-cli lpush transactions '{"timestamp":"2024-01-01T10:00:00","amount":100.50,"user_id":"user123","type":"purchase"}'
```

[elk-stack]: https://www.elastic.co/what-is/elk-stack
[elastic-docker]: https://www.docker.elastic.co/
[subscriptions]: https://www.elastic.co/subscriptions
[es-security]: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html
[license-settings]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html
[license-mngmt]: https://www.elastic.co/guide/en/kibana/current/managing-licenses.html
[license-apis]: https://www.elastic.co/guide/en/elasticsearch/reference/current/licensing-apis.html

[elastdocker]: https://github.com/sherifabdlnaby/elastdocker

[docker-install]: https://docs.docker.com/get-docker/
[compose-install]: https://docs.docker.com/compose/install/
[compose-v2]: https://docs.docker.com/compose/compose-v2/
[linux-postinstall]: https://docs.docker.com/engine/install/linux-postinstall/

[bootstrap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html
[es-heap]: https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings

[win-filesharing]: https://docs.docker.com/desktop/settings/windows/#file-sharing
[mac-filesharing]: https://docs.docker.com/desktop/settings/mac/#file-sharing

[builtin-users]: https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
[ls-monitoring]: https://www.elastic.co/guide/en/logstash/current/monitoring-with-metricbeat.html
[sec-cluster]: https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html