# Netty 클라이언트 라이브러리 구축 가이드

> **프로덕션 사례 기반 실전 가이드**
> Spring WebFlux와 Ktor의 실제 구현 분석을 통한 Netty 클라이언트 라이브러리 설계 및 구현

---

## 📋 목차

1. [소개](#1-소개)
2. [Netty 핵심 개념 복습](#2-netty-핵심-개념-복습)
3. [프로덕션 사례 분석](#3-프로덕션-사례-분석)
   - 3.1 [Spring WebFlux (Reactor Netty)](#31-spring-webflux-reactor-netty)
   - 3.2 [Ktor Netty Engine](#32-ktor-netty-engine)
4. [클라이언트 라이브러리 설계 패턴](#4-클라이언트-라이브러리-설계-패턴)
5. [실전 구현 가이드](#5-실전-구현-가이드)
6. [NETTY_분석_가이드.md 활용법](#6-netty_분석_가이드md-활용법)
7. [체크리스트 및 권장사항](#7-체크리스트-및-권장사항)
8. [참고 자료](#8-참고-자료)

---

## 1. 소개

### 1.1 이 문서의 목적

이 가이드는 **Netty를 활용한 클라이언트 라이브러리를 구축**하려는 개발자를 위한 실전 가이드입니다. 단순한 API 설명이 아닌, **Spring WebFlux와 Ktor 같은 프로덕션 급 프레임워크가 Netty를 어떻게 활용하는지 실제 코드를 분석**하여, 검증된 패턴과 Best Practice를 제시합니다.

### 1.2 대상 독자

- Netty 기반 HTTP/TCP 클라이언트 라이브러리를 개발하려는 개발자
- Spring WebFlux, Ktor 등의 프레임워크 내부 구조를 이해하고 싶은 개발자
- 고성능 비동기 네트워크 프로그래밍에 관심 있는 개발자

### 1.3 전제 조건

- Java 또는 Kotlin 기본 문법 이해
- 비동기 프로그래밍 개념 (Future, CompletableFuture, Coroutine 등)
- HTTP 프로토콜 기본 지식

> 💡 **추천**: Netty의 기초 개념이 생소하다면 먼저 `NETTY_분석_가이드.md`의 섹션 1-4를 읽고 오시길 권장합니다.

---

## 2. Netty 핵심 개념 복습

클라이언트 라이브러리 구현에 필요한 핵심 Netty 개념을 간단히 복습합니다.

### 2.1 핵심 컴포넌트

#### Bootstrap
**클라이언트 애플리케이션을 시작하는 헬퍼 클래스**

```java
// Netty 4.2 권장 방식
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(group)
         .channel(NioSocketChannel.class)
         .handler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline().addLast(new YourHandler());
             }
         });

ChannelFuture future = bootstrap.connect("example.com", 80).sync();
```

**주요 메소드**:
- `group()`: EventLoopGroup 설정
- `channel()`: 사용할 Channel 클래스 지정
- `handler()`: ChannelHandler 설정
- `option()`: ChannelOption 설정 (타임아웃, 버퍼 크기 등)

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 4.1 참조

#### EventLoopGroup / EventLoop
**이벤트를 처리하는 스레드 풀**

```java
// NIO Transport (범용)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());

// Epoll Transport (Linux 고성능)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(EpollIoHandler.newFactory());

// KQueue Transport (macOS/BSD)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(KQueueIoHandler.newFactory());
```

**핵심 특징**:
- 하나의 EventLoop = 하나의 Thread
- 각 Channel은 하나의 EventLoop에 바인딩
- Thread-safe하지 않은 로직도 EventLoop 내에서 안전하게 실행

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 4.3 참조

#### Channel & ChannelPipeline
**네트워크 소켓을 추상화한 인터페이스**

```java
ChannelPipeline pipeline = channel.pipeline();
pipeline.addLast("decoder", new HttpResponseDecoder());
pipeline.addLast("aggregator", new HttpObjectAggregator(8192));
pipeline.addLast("encoder", new HttpRequestEncoder());
pipeline.addLast("handler", new MyClientHandler());
```

**Pipeline 처리 순서**:
```
[Inbound]  네트워크 → Decoder → Aggregator → Handler → 애플리케이션
[Outbound] 애플리케이션 → Handler → Encoder → 네트워크
```

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 4.4, 4.5 참조

### 2.2 클라이언트 개발에 필수적인 개념

| 개념 | 설명 | 참조 섹션 |
|------|------|-----------|
| **ByteBuf** | Netty의 바이트 버퍼 (JDK ByteBuffer보다 효율적) | 섹션 5.1 |
| **ChannelFuture** | 비동기 작업 결과를 나타내는 Future | 섹션 4.2 |
| **ChannelHandler** | 이벤트 처리 로직을 구현하는 인터페이스 | 섹션 4.4 |
| **Codec** | Encoder/Decoder (프로토콜 변환) | 섹션 6 |

---

## 3. 프로덕션 사례 분석

실제 프로덕션 환경에서 사용되는 Spring WebFlux와 Ktor의 Netty 활용 방식을 분석합니다.

---

## 3.1 Spring WebFlux (Reactor Netty)

Spring WebFlux는 **Reactor Netty**를 기본 HTTP 클라이언트/서버로 사용합니다. Reactor Netty는 Netty를 Reactive Streams 방식으로 래핑한 고수준 라이브러리입니다.

### 3.1.1 아키텍처 개요

```
┌─────────────────────────────────────────────┐
│         Spring WebFlux Application          │
└─────────────────┬───────────────────────────┘
                  │ (Reactive API)
┌─────────────────▼───────────────────────────┐
│            Reactor Netty Client             │
│  ┌──────────────────────────────────────┐   │
│  │      HttpClient (High-Level API)     │   │
│  └──────────────┬───────────────────────┘   │
│  ┌──────────────▼───────────────────────┐   │
│  │   Bootstrap + EventLoopGroup Setup   │   │
│  ├──────────────────────────────────────┤   │
│  │     Connection Pool Management       │   │
│  ├──────────────────────────────────────┤   │
│  │    ChannelPipeline Configuration     │   │
│  └──────────────────────────────────────┘   │
└─────────────────┬───────────────────────────┘
                  │ (Netty Core API)
┌─────────────────▼───────────────────────────┐
│         Netty EventLoopGroup (NIO)          │
└─────────────────────────────────────────────┘
```

### 3.1.2 핵심 구현 분석

#### (1) HttpClient 생성 및 설정

```java
import reactor.netty.http.client.HttpClient;
import reactor.netty.resources.ConnectionProvider;
import io.netty.channel.ChannelOption;

// 기본 클라이언트 생성
HttpClient client = HttpClient.create();

// 커스텀 연결 풀 설정
ConnectionProvider provider = ConnectionProvider.builder("custom")
    .maxConnections(50)                          // 최대 연결 수
    .maxIdleTime(Duration.ofSeconds(20))         // 유휴 연결 타임아웃
    .maxLifeTime(Duration.ofSeconds(60))         // 연결 최대 수명
    .pendingAcquireTimeout(Duration.ofSeconds(60)) // 연결 대기 타임아웃
    .evictInBackground(Duration.ofSeconds(120))  // 백그라운드 정리 주기
    .metrics(true)                               // 메트릭 활성화
    .build();

HttpClient client = HttpClient.create(provider);
```

**핵심 패턴**: **Fluent Builder Pattern**으로 설정을 체이닝하여 가독성 향상

#### (2) EventLoopGroup 설정

```java
import reactor.netty.resources.LoopResources;

// 커스텀 EventLoop 리소스
LoopResources loop = LoopResources.create(
    "event-loop",  // 스레드 이름 prefix
    1,             // Selector 스레드 수 (일반적으로 1)
    4,             // Worker 스레드 수
    true           // Daemon 스레드 여부
);

HttpClient client = HttpClient.create()
    .runOn(loop);
```

**기본 동작**:
- Worker 스레드 수 = `Runtime.getRuntime().availableProcessors()` (최소 4)
- 시스템 속성으로 조정 가능: `-Dreactor.netty.ioWorkerCount=8`

#### (3) TCP 레벨 설정

```java
import io.netty.channel.ChannelOption;
import io.netty.channel.epoll.EpollChannelOption;

HttpClient client = HttpClient.create()
    // 바인딩 주소 (클라이언트 소켓의 로컬 주소)
    .bindAddress(() -> new InetSocketAddress("host", 1234))

    // 연결 타임아웃
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)

    // TCP KeepAlive 설정
    .option(ChannelOption.SO_KEEPALIVE, true)
    .option(EpollChannelOption.TCP_KEEPIDLE, 300)   // 300초 유휴 후 KeepAlive 시작
    .option(EpollChannelOption.TCP_KEEPINTVL, 60)   // 60초 간격으로 probe 전송
    .option(EpollChannelOption.TCP_KEEPCNT, 8);     // 8번 실패 시 연결 종료
```

**중요 ChannelOption**:

| 옵션 | 설명 | 권장값 |
|------|------|--------|
| `CONNECT_TIMEOUT_MILLIS` | 연결 수립 타임아웃 | 5000-10000ms |
| `SO_KEEPALIVE` | TCP KeepAlive 활성화 | true |
| `TCP_NODELAY` | Nagle 알고리즘 비활성화 (지연 감소) | true |
| `SO_REUSEADDR` | TIME_WAIT 상태 소켓 재사용 | true |

#### (4) ChannelPipeline 확장

```java
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.logging.LoggingHandler;
import java.util.concurrent.TimeUnit;

HttpClient client = HttpClient.create()
    // 연결 후 핸들러 추가
    .doOnConnected(conn ->
        conn.addHandlerFirst(new ReadTimeoutHandler(10, TimeUnit.SECONDS))
    )

    // 채널 초기화 시 핸들러 추가
    .doOnChannelInit((observer, channel, remoteAddress) ->
        channel.pipeline()
               .addFirst(new LoggingHandler("reactor.netty.client"))
    );
```

**Lifecycle Callback 정리**:

| Callback | 실행 시점 | 용도 |
|----------|-----------|------|
| `doOnChannelInit` | 채널 생성 직후 | Pipeline 초기 설정 |
| `doOnConnect` | 연결 시도 전 | 연결 전 준비 작업 |
| `doOnConnected` | 연결 성공 후 | 추가 핸들러 등록, 타임아웃 설정 |
| `doOnRequest` | 요청 전송 전 | 요청 로깅, 헤더 수정 |
| `doOnResponse` | 응답 수신 후 | 응답 헤더 검증, 로깅 |
| `doOnError` | 에러 발생 시 | 에러 핸들링, 재시도 로직 |

#### (5) 요청 전송 및 응답 처리

```java
import reactor.core.publisher.Mono;

// GET 요청
Mono<String> response = client.get()
    .uri("https://example.com/api/data")
    .responseContent()        // 응답 바디 스트림
    .aggregate()              // 전체 바디를 하나의 ByteBuf로 합침
    .asString();              // String으로 변환

// POST 요청
client.post()
    .uri("https://example.com/api/data")
    .send(ByteBufFlux.fromString(Mono.just("{\"key\":\"value\"}")))
    .responseSingle((resp, bytes) -> {
        System.out.println("Status: " + resp.status());
        System.out.println("Headers: " + resp.responseHeaders());
        return bytes.asString();
    })
    .block();
```

**핵심 API**:
- `responseContent()`: 응답 바디를 `ByteBufFlux`로 반환
- `responseSingle()`: 응답 메타데이터와 바디를 함께 처리
- `aggregate()`: 스트리밍 데이터를 하나의 ByteBuf로 수집

#### (6) 연결 풀 관리

```java
// 연결 풀 비활성화 (매 요청마다 새 연결)
HttpClient client = HttpClient.newConnection()
    .doOnConnected(conn ->
        System.out.println("New connection: " + conn.channel()));

// 기본 연결 풀 설정
// - 최대 500개 활성 연결
// - 1000개의 대기 중인 획득 시도
```

**연결 풀 동작 방식**:
1. 요청 발생 시 풀에서 유휴 연결 검색
2. 없으면 새 연결 생성 (최대 개수 제한)
3. 모두 사용 중이면 대기 큐에 추가
4. 연결 반환 시 자동으로 풀로 회수

### 3.1.3 주요 설계 패턴

#### 패턴 1: Reactive Streams 통합

```java
// Mono: 0..1개 결과 (단일 응답)
Mono<String> response = client.get()
    .uri("/api/user/123")
    .responseContent()
    .aggregate()
    .asString();

// Flux: 0..N개 결과 (스트리밍)
Flux<String> stream = client.get()
    .uri("/api/events")
    .responseContent()
    .asString();

stream.subscribe(
    data -> System.out.println("Received: " + data),
    error -> System.err.println("Error: " + error),
    () -> System.out.println("Stream completed")
);
```

**장점**:
- Backpressure 자동 처리
- 함수형 프로그래밍 스타일
- Spring WebFlux와 원활한 통합

#### 패턴 2: 리소스 라이프사이클 관리

```java
// 전역 리소스 사용 (권장)
HttpClient client = HttpClient.create();
// 자동으로 HttpResources.get() 사용

// 종료 시 정리
HttpResources.disposeLoopsAndConnections();

// 커스텀 리소스 사용
ConnectionProvider provider = ConnectionProvider.create("custom", 10);
LoopResources loop = LoopResources.create("event-loop", 4, true);

HttpClient client = HttpClient.create(provider)
    .runOn(loop);

// 사용 후 명시적 해제
provider.disposeLater().block();
loop.disposeLater().block();
```

**핵심 원칙**:
- 가능하면 전역 공유 리소스 사용 (HttpResources)
- 커스텀 리소스는 명시적으로 해제
- SDK가 라이프사이클을 관리하도록 설계

#### 패턴 3: 메트릭 및 모니터링

```java
HttpClient client = HttpClient.create()
    .metrics(true, uriPath -> {
        // URI 정규화 (동적 파라미터 제거)
        if (uriPath.startsWith("/api/user/")) {
            return "/api/user/{id}";
        }
        return uriPath;
    });
```

**수집 가능한 메트릭**:
- `reactor.netty.http.client.data.received/sent`: 데이터 전송량
- `reactor.netty.http.client.errors`: 에러 발생 횟수
- `reactor.netty.http.client.connect.time`: 연결 수립 시간
- `reactor.netty.http.client.response.time`: 응답 시간
- Connection Pool 메트릭 (활성, 유휴, 대기 연결 수)

---

## 3.2 Ktor Netty Engine

Ktor는 Kotlin으로 작성된 비동기 웹 프레임워크로, 서버/클라이언트 모두 Netty를 지원합니다. 여기서는 **서버 사이드** Netty Engine 구현을 분석합니다.

### 3.2.1 아키텍처 개요

```
┌─────────────────────────────────────────────┐
│          Ktor Application (Kotlin)          │
│         (Routing, Features, etc.)           │
└─────────────────┬───────────────────────────┘
                  │ (Coroutine API)
┌─────────────────▼───────────────────────────┐
│       NettyApplicationEngine (Ktor)         │
│  ┌──────────────────────────────────────┐   │
│  │   connectionEventGroup (Boss)        │   │
│  │   workerEventGroup (Worker)          │   │
│  │   callEventGroup (Pipeline Executor) │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │     ServerBootstrap Configuration    │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │  NettyChannelInitializer (Pipeline)  │   │
│  └──────────────────────────────────────┘   │
└─────────────────┬───────────────────────────┘
                  │ (Netty Core API)
┌─────────────────▼───────────────────────────┐
│      Netty EventLoopGroup (NIO/Epoll)       │
└─────────────────────────────────────────────┘
```

### 3.2.2 핵심 구현 분석

#### (1) EventLoopGroup 구성

Ktor는 **3개의 EventLoopGroup**을 사용하여 역할을 분리합니다:

```kotlin
// 의사 코드 (실제 Ktor 내부 구조)
class NettyApplicationEngine {
    // Boss Group: 클라이언트 연결 수락
    private val connectionEventGroup: EventLoopGroup by lazy {
        createEventGroup(configuration.connectionGroupSize)
    }

    // Worker Group: HTTP 요청 처리 및 엔진 내부 작업
    private val workerEventGroup: EventLoopGroup by lazy {
        createEventGroup(configuration.workerGroupSize)
    }

    // Call Group: 파이프라인 호출 실행 (선택적)
    private val callEventGroup: EventLoopGroup by lazy {
        if (configuration.shareWorkGroup) {
            workerEventGroup  // Worker Group 공유
        } else {
            createEventGroup(configuration.callGroupSize)
        }
    }
}
```

**EventLoopGroup 역할**:

| Group | 역할 | 기본 크기 | 설명 |
|-------|------|-----------|------|
| **connectionEventGroup** | Connection Acceptor | `parallelism / 2` | 클라이언트 연결 수락 (ServerSocketChannel) |
| **workerEventGroup** | Request Handler | `parallelism` | HTTP 요청/응답 처리 (SocketChannel) |
| **callEventGroup** | Pipeline Executor | `parallelism` | Ktor 애플리케이션 파이프라인 실행 |

**메모리 최적화**:
- `shareWorkGroup = true`: Worker와 Call을 동일 그룹 사용 (메모리 절약)
- `shareWorkGroup = false`: 별도 그룹 사용 (격리된 실행)

#### (2) Channel 클래스 선택

```kotlin
fun getChannelClass(): KClass<out ServerSocketChannel> = when {
    KQueue.isAvailable() -> KQueueServerSocketChannel::class  // macOS/BSD
    Epoll.isAvailable() -> EpollServerSocketChannel::class     // Linux
    else -> NioServerSocketChannel::class                      // 기본 (범용)
}
```

**자동 최적화**:
- 플랫폼별 최적 Transport 자동 선택
- KQueue (macOS) > Epoll (Linux) > NIO (범용)
- 명시적 설정 없이 최고 성능 달성

#### (3) ServerBootstrap 설정

```kotlin
private fun createBootstrap(): ServerBootstrap {
    val bootstrap = ServerBootstrap()

    // EventLoopGroup 설정
    bootstrap.group(connectionEventGroup, workerEventGroup)

    // Channel 클래스 설정
    bootstrap.channel(getChannelClass().java)

    // TCP 옵션 설정
    bootstrap.childOption(ChannelOption.TCP_NODELAY, true)
    bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true)

    // ChannelPipeline 초기화
    bootstrap.childHandler(NettyChannelInitializer(
        enginePipeline,
        applicationProvider,
        callEventGroup,
        userContext
    ))

    return bootstrap
}
```

**주요 설정 항목**:
- `group(parent, child)`: Boss와 Worker EventLoopGroup 분리
- `childOption()`: 각 클라이언트 소켓에 적용될 옵션
- `childHandler()`: 새 연결마다 실행될 ChannelInitializer

#### (4) Configuration 옵션

```kotlin
// Ktor 서버 설정 예제
embeddedServer(Netty, port = 8080) {
    // Netty 특화 설정
}.apply {
    // NettyApplicationEngine.Configuration 접근
    (this as NettyApplicationEngine).configuration.apply {
        // 스레드 풀 크기
        connectionGroupSize = 2
        workerGroupSize = 8
        callGroupSize = 8
        shareWorkGroup = false

        // HTTP 설정
        requestQueueLimit = 32           // 파이프라인당 동시 요청 수
        responseWriteTimeoutSeconds = 10 // 응답 쓰기 타임아웃
        requestReadTimeoutSeconds = 0    // 요청 읽기 타임아웃 (0 = 무제한)

        // HTTP 코덱 제한
        maxInitialLineLength = 4096      // 첫 줄 최대 길이
        maxHeaderSize = 8192             // 헤더 최대 크기
        maxChunkSize = 8192              // Chunk 최대 크기

        // HTTP/2 지원
        enableHttp2 = true
        enableH2c = false                // HTTP/2 Cleartext (TLS 없이)

        // Bootstrap 커스터마이징
        configureBootstrap = { bootstrap ->
            bootstrap.option(ChannelOption.SO_BACKLOG, 128)
        }
    }
}
```

**핵심 설정 가이드**:

| 설정 | 설명 | 권장값 | 주의사항 |
|------|------|--------|----------|
| `connectionGroupSize` | Boss 스레드 수 | 1-2 | 대부분 1-2면 충분 |
| `workerGroupSize` | Worker 스레드 수 | CPU 코어 수 | CPU 바운드 작업 많으면 더 증가 |
| `shareWorkGroup` | Worker/Call 그룹 공유 | true | 메모리 절약, 격리 필요 시 false |
| `requestQueueLimit` | 파이프라인 동시 요청 | 32 | 높을수록 처리량 증가, 메모리 증가 |
| `enableHttp2` | HTTP/2 활성화 | true (TLS 사용 시) | HTTP/1.1 호환성 유지 |

#### (5) 라이프사이클 관리

```kotlin
// 시작
fun start() {
    // 각 Connector(포트)마다 Bootstrap 생성 및 바인딩
    connectors.forEach { connector ->
        val bootstrap = createBootstrap()
        val channelFuture = bootstrap.bind(connector.host, connector.port).sync()
        channels.add(channelFuture.channel())
    }
}

// 종료
fun stop(gracePeriodMillis: Long, timeoutMillis: Long) {
    // 1. 모든 채널 닫기
    channels.forEach { it.close().await() }

    // 2. EventLoopGroup 종료
    connectionEventGroup.shutdownGracefully(
        gracePeriodMillis, timeoutMillis, TimeUnit.MILLISECONDS
    ).await()

    workerEventGroup.shutdownGracefully(
        gracePeriodMillis, timeoutMillis, TimeUnit.MILLISECONDS
    ).await()

    // 3. Call Group 종료 (공유되지 않은 경우만)
    if (!configuration.shareWorkGroup) {
        callEventGroup.shutdownGracefully(
            gracePeriodMillis, timeoutMillis, TimeUnit.MILLISECONDS
        ).await()
    }
}
```

**Graceful Shutdown 과정**:
1. **새 연결 거부**: ServerSocketChannel 닫기
2. **기존 요청 완료 대기**: `gracePeriodMillis` 동안 진행 중인 요청 완료 허용
3. **강제 종료**: `timeoutMillis` 초과 시 강제 종료

> ⚠️ **주의**: Ktor는 "모든 요청 완료 대기"를 하지만, Netty EventLoopGroup은 `gracePeriod` 동안 새 태스크를 계속 받아들입니다. 따라서 별도의 타임아웃 계산이 필요합니다.

#### (6) Coroutine 통합

```kotlin
// EventLoopGroup을 Coroutine Dispatcher로 변환
val dispatcher = workerEventGroup.asCoroutineDispatcher()

// Ktor Application에서 사용
routing {
    get("/api/data") {
        withContext(dispatcher) {
            // Netty EventLoop에서 실행
            val data = fetchData()
            call.respond(data)
        }
    }
}
```

**핵심 개념**:
- `asCoroutineDispatcher()`: EventLoopGroup → CoroutineDispatcher 변환
- Suspend 함수가 Netty EventLoop 스레드에서 실행
- Thread Context Switching 최소화

### 3.2.3 주요 설계 패턴

#### 패턴 1: 역할 기반 EventLoopGroup 분리

```
┌─────────────────────────────────────────┐
│      connectionEventGroup (Boss)        │ ← 새 연결 수락만 담당
│               (1-2 threads)             │
└────────────┬────────────────────────────┘
             │ 새 연결 전달
┌────────────▼────────────────────────────┐
│       workerEventGroup (Worker)         │ ← I/O 처리 (읽기/쓰기)
│          (CPU cores threads)            │
└────────────┬────────────────────────────┘
             │ 요청 전달
┌────────────▼────────────────────────────┐
│      callEventGroup (Pipeline)          │ ← 비즈니스 로직 실행
│          (CPU cores threads)            │
└─────────────────────────────────────────┘
```

**장점**:
- Boss는 연결 수락만 → 높은 처리량
- Worker는 I/O만 → Non-blocking 유지
- Call은 무거운 작업 허용 → Worker 블로킹 방지

#### 패턴 2: Lazy Initialization

```kotlin
// 실제 사용 전까지 EventLoopGroup 생성 지연
private val connectionEventGroup: EventLoopGroup by lazy {
    createEventGroup(configuration.connectionGroupSize)
}
```

**장점**:
- 사용하지 않는 리소스는 생성하지 않음
- 초기 시작 시간 단축
- 메모리 절약

#### 패턴 3: Platform-Aware Channel Selection

```kotlin
when {
    KQueue.isAvailable() -> KQueueServerSocketChannel::class  // macOS 최적화
    Epoll.isAvailable() -> EpollServerSocketChannel::class     // Linux 최적화
    else -> NioServerSocketChannel::class                      // 범용
}
```

**장점**:
- 플랫폼별 최고 성능 자동 선택
- 개발자가 신경 쓸 필요 없음
- 이식성 유지

---

## 4. 클라이언트 라이브러리 설계 패턴

Spring WebFlux와 Ktor의 사례를 바탕으로, 클라이언트 라이브러리 설계 시 적용해야 할 핵심 패턴을 정리합니다.

### 4.1 핵심 설계 원칙

#### 원칙 1: 리소스 라이프사이클 명확화

**BAD** ❌:
```java
public class MyClient {
    public Response request(String url) {
        EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
        Bootstrap bootstrap = new Bootstrap();
        // ... 요청 처리
        group.shutdownGracefully();  // 매 요청마다 생성/종료 → 비효율
        return response;
    }
}
```

**GOOD** ✅:
```java
public class MyClient implements AutoCloseable {
    private final EventLoopGroup group;
    private final Bootstrap bootstrap;

    public MyClient() {
        // 한 번만 생성
        this.group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
        this.bootstrap = new Bootstrap()
            .group(group)
            .channel(NioSocketChannel.class);
    }

    public Response request(String url) {
        // Bootstrap 재사용
        return bootstrap.connect(...).sync();
    }

    @Override
    public void close() {
        // 종료 시 한 번만 정리
        group.shutdownGracefully().await();
    }
}
```

#### 원칙 2: Fluent Builder API 제공

**참조**: Reactor Netty의 `HttpClient.create()` 패턴

```java
MyClient client = MyClient.builder()
    .connectTimeout(Duration.ofSeconds(5))
    .readTimeout(Duration.ofSeconds(10))
    .maxConnections(50)
    .enableMetrics(true)
    .build();
```

**장점**:
- 가독성 높음
- 선택적 설정 명확
- Immutable 객체 생성 가능

#### 원칙 3: Connection Pool 구현

**BAD** ❌:
```java
// 매번 새 연결 생성
Channel channel = bootstrap.connect(host, port).sync().channel();
```

**GOOD** ✅:
```java
public class ConnectionPool {
    private final ConcurrentLinkedQueue<Channel> pool = new ConcurrentLinkedQueue<>();
    private final AtomicInteger activeCount = new AtomicInteger(0);
    private final int maxConnections;

    public Channel acquire() {
        Channel channel = pool.poll();
        if (channel != null && channel.isActive()) {
            return channel;
        }

        if (activeCount.get() < maxConnections) {
            activeCount.incrementAndGet();
            return createNewConnection();
        }

        // 대기 또는 예외
        throw new ConnectionPoolExhaustedException();
    }

    public void release(Channel channel) {
        if (channel.isActive()) {
            pool.offer(channel);
        } else {
            activeCount.decrementAndGet();
        }
    }
}
```

#### 원칙 4: Timeout 계층화

```java
MyClient client = MyClient.builder()
    .connectTimeout(Duration.ofSeconds(5))      // 연결 수립 타임아웃
    .requestTimeout(Duration.ofSeconds(10))     // 전체 요청 타임아웃
    .readTimeout(Duration.ofSeconds(30))        // 읽기 유휴 타임아웃
    .writeTimeout(Duration.ofSeconds(30))       // 쓰기 유휴 타임아웃
    .build();
```

**Timeout 적용 위치**:
- **Connect Timeout**: `ChannelOption.CONNECT_TIMEOUT_MILLIS`
- **Read Timeout**: `ReadTimeoutHandler`
- **Write Timeout**: `WriteTimeoutHandler`
- **Request Timeout**: 애플리케이션 레벨 (ScheduledFuture)

#### 원칙 5: 비동기 API 우선 설계

**Callback 방식** (Netty 네이티브):
```java
client.requestAsync(url, new Callback() {
    @Override
    public void onSuccess(Response response) {
        // 성공 처리
    }

    @Override
    public void onFailure(Throwable error) {
        // 에러 처리
    }
});
```

**Future 방식** (CompletableFuture):
```java
CompletableFuture<Response> future = client.requestAsync(url);

future.thenAccept(response -> {
    // 성공 처리
}).exceptionally(error -> {
    // 에러 처리
    return null;
});
```

**Reactive 방식** (Reactor, RxJava):
```java
Mono<Response> mono = client.request(url);

mono.subscribe(
    response -> { /* 성공 */ },
    error -> { /* 에러 */ }
);
```

> 💡 **권장**: 비동기를 기본으로, 동기 API는 비동기 위에 래퍼로 제공
> ```java
> public Response requestSync(String url) {
>     return requestAsync(url).get();  // 내부적으로 비동기 호출
> }
> ```

---

### 4.2 ChannelPipeline 설계

#### 표준 HTTP 클라이언트 Pipeline

```
[Outbound: 요청 전송]
Application
    ↓
RequestTimeoutHandler       // 전체 요청 타임아웃
    ↓
HttpRequestEncoder          // HTTP 메시지 → 바이트
    ↓
WriteTimeoutHandler         // 쓰기 타임아웃
    ↓
Network

[Inbound: 응답 수신]
Network
    ↓
ReadTimeoutHandler          // 읽기 타임아웃
    ↓
HttpResponseDecoder         // 바이트 → HTTP 메시지
    ↓
HttpObjectAggregator        // Chunked 메시지 합치기
    ↓
ContentDecompressor         // Gzip 압축 해제 (선택)
    ↓
ResponseHandler             // 응답 처리
    ↓
Application
```

#### 구현 예제

```java
bootstrap.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();

        // Outbound (요청 방향)
        p.addLast("encoder", new HttpRequestEncoder());
        p.addLast("writeTimeout", new WriteTimeoutHandler(30, TimeUnit.SECONDS));

        // Inbound (응답 방향)
        p.addLast("readTimeout", new ReadTimeoutHandler(30, TimeUnit.SECONDS));
        p.addLast("decoder", new HttpResponseDecoder());
        p.addLast("aggregator", new HttpObjectAggregator(10 * 1024 * 1024)); // 10MB
        p.addLast("decompressor", new HttpContentDecompressor());
        p.addLast("handler", new MyResponseHandler());
    }
});
```

**Handler 순서 규칙**:
- Inbound: 네트워크에 가까운 순서대로 (Decoder → Aggregator → Handler)
- Outbound: 반대 순서로 자동 실행 (Handler → Encoder → Network)

---

### 4.3 에러 처리 전략

#### 레벨 1: Channel 레벨 에러

```java
public class ErrorHandlingHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        if (cause instanceof ReadTimeoutException) {
            // 읽기 타임아웃
            logger.warn("Read timeout: {}", ctx.channel());
        } else if (cause instanceof WriteTimeoutException) {
            // 쓰기 타임아웃
            logger.warn("Write timeout: {}", ctx.channel());
        } else if (cause instanceof IOException) {
            // 네트워크 에러
            logger.error("Network error: {}", cause.getMessage());
        } else {
            // 기타 에러
            logger.error("Unexpected error", cause);
        }

        ctx.close();  // 채널 닫기
    }
}
```

#### 레벨 2: 요청 레벨 에러

```java
CompletableFuture<Response> future = new CompletableFuture<>();

channel.writeAndFlush(request).addListener((ChannelFutureListener) f -> {
    if (!f.isSuccess()) {
        // 쓰기 실패
        future.completeExceptionally(new RequestFailedException(f.cause()));
    }
});

// 응답 핸들러에서 완료
public class ResponseHandler extends SimpleChannelInboundHandler<FullHttpResponse> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpResponse msg) {
        if (msg.status().code() >= 400) {
            future.completeExceptionally(new HttpErrorException(msg.status()));
        } else {
            future.complete(new Response(msg));
        }
    }
}
```

#### 레벨 3: 재시도 로직

```java
public class RetryPolicy {
    private final int maxRetries;
    private final Duration backoff;

    public <T> CompletableFuture<T> execute(Supplier<CompletableFuture<T>> action) {
        return executeWithRetry(action, 0);
    }

    private <T> CompletableFuture<T> executeWithRetry(
            Supplier<CompletableFuture<T>> action,
            int attempt) {

        return action.get().exceptionally(error -> {
            if (attempt < maxRetries && isRetryable(error)) {
                // 지수 백오프
                long delayMs = (long) (backoff.toMillis() * Math.pow(2, attempt));

                return CompletableFuture.delayedExecutor(
                    delayMs, TimeUnit.MILLISECONDS
                ).execute(() -> executeWithRetry(action, attempt + 1));
            }

            throw new CompletionException(error);
        });
    }

    private boolean isRetryable(Throwable error) {
        return error instanceof ConnectException
            || error instanceof ReadTimeoutException
            || (error instanceof HttpErrorException
                && ((HttpErrorException) error).getStatus() == 503);
    }
}
```

---

### 4.4 메트릭 및 모니터링

#### 핵심 메트릭

```java
public class ClientMetrics {
    // Connection Pool
    private final AtomicInteger activeConnections = new AtomicInteger(0);
    private final AtomicInteger idleConnections = new AtomicInteger(0);
    private final AtomicInteger pendingAcquires = new AtomicInteger(0);

    // Request
    private final LongAdder totalRequests = new LongAdder();
    private final LongAdder successfulRequests = new LongAdder();
    private final LongAdder failedRequests = new LongAdder();

    // Timing
    private final Histogram responseTime = new Histogram();
    private final Histogram connectTime = new Histogram();

    // Throughput
    private final LongAdder bytesSent = new LongAdder();
    private final LongAdder bytesReceived = new LongAdder();
}
```

#### 메트릭 수집 Handler

```java
public class MetricsHandler extends ChannelDuplexHandler {
    private final ClientMetrics metrics;
    private long requestStartTime;

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        requestStartTime = System.nanoTime();

        if (msg instanceof ByteBuf) {
            metrics.bytesSent.add(((ByteBuf) msg).readableBytes());
        }

        ctx.write(msg, promise);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        long duration = System.nanoTime() - requestStartTime;
        metrics.responseTime.update(duration / 1_000_000);  // ms

        if (msg instanceof ByteBuf) {
            metrics.bytesReceived.add(((ByteBuf) msg).readableBytes());
        }

        if (msg instanceof FullHttpResponse) {
            FullHttpResponse response = (FullHttpResponse) msg;
            if (response.status().code() < 400) {
                metrics.successfulRequests.increment();
            } else {
                metrics.failedRequests.increment();
            }
        }

        ctx.fireChannelRead(msg);
    }
}
```

---

## 5. 실전 구현 가이드

이제 실제로 Netty 기반 HTTP 클라이언트 라이브러리를 구현해보겠습니다.

### 5.1 기본 클라이언트 구현

```java
package com.example.netty.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioIoHandler;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.http.*;
import io.netty.handler.timeout.ReadTimeoutHandler;

import java.net.URI;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class SimpleHttpClient implements AutoCloseable {

    private final EventLoopGroup eventLoopGroup;
    private final Bootstrap bootstrap;
    private final Duration connectTimeout;
    private final Duration readTimeout;

    private SimpleHttpClient(Builder builder) {
        this.connectTimeout = builder.connectTimeout;
        this.readTimeout = builder.readTimeout;

        // EventLoopGroup 생성
        this.eventLoopGroup = new MultiThreadIoEventLoopGroup(
            builder.workerThreads,
            NioIoHandler.newFactory()
        );

        // Bootstrap 설정
        this.bootstrap = new Bootstrap();
        bootstrap.group(eventLoopGroup)
                 .channel(NioSocketChannel.class)
                 .option(ChannelOption.CONNECT_TIMEOUT_MILLIS,
                         (int) connectTimeout.toMillis())
                 .option(ChannelOption.SO_KEEPALIVE, true)
                 .option(ChannelOption.TCP_NODELAY, true);
    }

    /**
     * GET 요청 (비동기)
     */
    public CompletableFuture<HttpResponse> getAsync(String url) {
        return requestAsync(HttpMethod.GET, url, null);
    }

    /**
     * POST 요청 (비동기)
     */
    public CompletableFuture<HttpResponse> postAsync(String url, String body) {
        return requestAsync(HttpMethod.POST, url, body);
    }

    /**
     * 공통 요청 로직
     */
    private CompletableFuture<HttpResponse> requestAsync(
            HttpMethod method, String url, String body) {

        CompletableFuture<HttpResponse> future = new CompletableFuture<>();

        try {
            URI uri = new URI(url);
            String host = uri.getHost();
            int port = uri.getPort() == -1 ? 80 : uri.getPort();
            String path = uri.getRawPath() +
                         (uri.getRawQuery() != null ? "?" + uri.getRawQuery() : "");

            // 연결 및 요청
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    ChannelPipeline p = ch.pipeline();

                    // Timeout
                    p.addLast("readTimeout",
                              new ReadTimeoutHandler(readTimeout.toSeconds(),
                                                     TimeUnit.SECONDS));

                    // HTTP Codec
                    p.addLast("decoder", new HttpResponseDecoder());
                    p.addLast("encoder", new HttpRequestEncoder());
                    p.addLast("aggregator",
                              new HttpObjectAggregator(10 * 1024 * 1024)); // 10MB

                    // Response Handler
                    p.addLast("handler", new ResponseHandler(future));
                }
            });

            // 연결
            ChannelFuture connectFuture = bootstrap.connect(host, port);

            connectFuture.addListener((ChannelFutureListener) cf -> {
                if (cf.isSuccess()) {
                    // HTTP 요청 생성
                    DefaultFullHttpRequest request = new DefaultFullHttpRequest(
                        HttpVersion.HTTP_1_1,
                        method,
                        path
                    );
                    request.headers().set(HttpHeaderNames.HOST, host);
                    request.headers().set(HttpHeaderNames.CONNECTION,
                                          HttpHeaderValues.CLOSE);
                    request.headers().set(HttpHeaderNames.ACCEPT_ENCODING,
                                          HttpHeaderValues.GZIP);

                    if (body != null) {
                        byte[] bytes = body.getBytes();
                        request.content().writeBytes(bytes);
                        request.headers().set(HttpHeaderNames.CONTENT_LENGTH, bytes.length);
                    }

                    // 요청 전송
                    cf.channel().writeAndFlush(request).addListener(wf -> {
                        if (!wf.isSuccess()) {
                            future.completeExceptionally(wf.cause());
                        }
                    });
                } else {
                    future.completeExceptionally(cf.cause());
                }
            });

        } catch (Exception e) {
            future.completeExceptionally(e);
        }

        return future;
    }

    /**
     * 동기 GET 요청 (내부적으로 비동기 호출)
     */
    public HttpResponse get(String url) {
        try {
            return getAsync(url).get();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void close() {
        eventLoopGroup.shutdownGracefully();
    }

    // ========== Response Handler ==========

    private static class ResponseHandler
            extends SimpleChannelInboundHandler<FullHttpResponse> {

        private final CompletableFuture<HttpResponse> future;

        public ResponseHandler(CompletableFuture<HttpResponse> future) {
            this.future = future;
        }

        @Override
        protected void channelRead0(ChannelHandlerContext ctx,
                                     FullHttpResponse msg) {
            // 응답 변환
            HttpResponse response = new HttpResponse(
                msg.status().code(),
                msg.content().toString(io.netty.util.CharsetUtil.UTF_8)
            );

            future.complete(response);
            ctx.close();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            future.completeExceptionally(cause);
            ctx.close();
        }
    }

    // ========== Builder ==========

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private int workerThreads = Runtime.getRuntime().availableProcessors();
        private Duration connectTimeout = Duration.ofSeconds(5);
        private Duration readTimeout = Duration.ofSeconds(30);

        public Builder workerThreads(int threads) {
            this.workerThreads = threads;
            return this;
        }

        public Builder connectTimeout(Duration timeout) {
            this.connectTimeout = timeout;
            return this;
        }

        public Builder readTimeout(Duration timeout) {
            this.readTimeout = timeout;
            return this;
        }

        public SimpleHttpClient build() {
            return new SimpleHttpClient(this);
        }
    }
}

// ========== Response DTO ==========

class HttpResponse {
    private final int statusCode;
    private final String body;

    public HttpResponse(int statusCode, String body) {
        this.statusCode = statusCode;
        this.body = body;
    }

    public int getStatusCode() { return statusCode; }
    public String getBody() { return body; }

    public boolean isSuccess() { return statusCode >= 200 && statusCode < 300; }
}
```

### 5.2 사용 예제

```java
public class Main {
    public static void main(String[] args) {
        // 클라이언트 생성
        SimpleHttpClient client = SimpleHttpClient.builder()
            .workerThreads(4)
            .connectTimeout(Duration.ofSeconds(5))
            .readTimeout(Duration.ofSeconds(10))
            .build();

        try {
            // 동기 요청
            HttpResponse response = client.get("http://example.com/api/data");
            System.out.println("Status: " + response.getStatusCode());
            System.out.println("Body: " + response.getBody());

            // 비동기 요청
            client.getAsync("http://example.com/api/data")
                  .thenAccept(resp -> {
                      System.out.println("Async response: " + resp.getBody());
                  })
                  .exceptionally(error -> {
                      System.err.println("Error: " + error.getMessage());
                      return null;
                  });

            // 비동기 완료 대기
            Thread.sleep(2000);

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            client.close();
        }
    }
}
```

### 5.3 Connection Pool 추가

```java
package com.example.netty.client;

import io.netty.channel.Channel;
import io.netty.channel.pool.*;

import java.net.InetSocketAddress;
import java.util.concurrent.TimeUnit;

public class PooledHttpClient extends SimpleHttpClient {

    private final ChannelPoolMap<InetSocketAddress, SimpleChannelPool> poolMap;

    public PooledHttpClient(Builder builder) {
        super(builder);

        this.poolMap = new AbstractChannelPoolMap<InetSocketAddress, SimpleChannelPool>() {
            @Override
            protected SimpleChannelPool newPool(InetSocketAddress key) {
                return new SimpleChannelPool(
                    bootstrap.remoteAddress(key),
                    new AbstractChannelPoolHandler() {
                        @Override
                        public void channelCreated(Channel ch) {
                            // 채널 초기화 (Pipeline 설정)
                            initChannel(ch);
                        }
                    }
                );
            }
        };
    }

    /**
     * 연결 획득
     */
    protected Channel acquireChannel(String host, int port) throws Exception {
        InetSocketAddress address = new InetSocketAddress(host, port);
        SimpleChannelPool pool = poolMap.get(address);
        return pool.acquire().get(connectTimeout.toMillis(), TimeUnit.MILLISECONDS);
    }

    /**
     * 연결 반환
     */
    protected void releaseChannel(Channel channel) {
        InetSocketAddress address = (InetSocketAddress) channel.remoteAddress();
        SimpleChannelPool pool = poolMap.get(address);
        pool.release(channel);
    }

    @Override
    public void close() {
        poolMap.close();
        super.close();
    }
}
```

---

## 6. NETTY_분석_가이드.md 활용법

이 프로젝트의 `NETTY_분석_가이드.md`는 Netty의 핵심 개념을 상세히 다루고 있습니다. 클라이언트 라이브러리 개발 시 다음과 같이 참고하세요.

### 6.1 학습 순서 가이드

#### Phase 1: Netty 기초 이해

| 섹션 | 내용 | 클라이언트 개발 시 활용 |
|------|------|------------------------|
| **1. Netty 소개** | Netty의 목적과 특징 | 왜 Netty를 사용해야 하는지 이해 |
| **2. 핵심 개념** | Event-Driven, Reactor 패턴 | 비동기 아키텍처 설계 기초 |
| **3. 아키텍처** | 전체 구조 파악 | 컴포넌트 간 관계 이해 |

**실습**: `NETTY_분석_가이드.md` 섹션 1-3 읽기 → Echo 서버 예제 실행

#### Phase 2: 핵심 컴포넌트 마스터

| 섹션 | 내용 | 클라이언트 개발 시 활용 |
|------|------|------------------------|
| **4.1 Bootstrap** | 클라이언트/서버 시작 | Bootstrap 설정 방법 학습 |
| **4.2 ChannelFuture** | 비동기 작업 처리 | 비동기 API 설계 |
| **4.3 EventLoop** | 스레드 모델 | EventLoopGroup 크기 결정, Thread-safe 이해 |
| **4.4 ChannelHandler** | 이벤트 처리 | 커스텀 Handler 작성 |
| **4.5 ChannelPipeline** | Handler 체인 | Pipeline 설계 및 순서 결정 |

**실습**: 이 가이드의 [5.1 기본 클라이언트 구현](#51-기본-클라이언트-구현) 코드를 따라 작성

#### Phase 3: 고급 기능 적용

| 섹션 | 내용 | 클라이언트 개발 시 활용 |
|------|------|------------------------|
| **5. ByteBuf** | 버퍼 관리 | 메모리 누수 방지, 효율적인 버퍼 사용 |
| **6. Codec** | 프로토콜 변환 | HTTP/Custom 프로토콜 Encoder/Decoder 작성 |
| **7. 예제 분석** | 실제 서버 코드 | Echo, HTTP 서버 구조 학습 |

**실습**: Connection Pool 추가, 재시도 로직 구현

#### Phase 4: 마이그레이션 (Netty 4.2)

| 섹션 | 내용 | 클라이언트 개발 시 활용 |
|------|------|------------------------|
| **4.3.3 마이그레이션 가이드** | NioEventLoopGroup → MultiThreadIoEventLoopGroup | 최신 API 사용 |
| **11. 마이그레이션 가이드** | 전체 체크리스트 | 기존 코드 업그레이드 |

### 6.2 섹션별 핵심 요약

#### 섹션 4.1: Bootstrap (클라이언트 시작)

**핵심 코드**:
```java
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(eventLoopGroup)                     // EventLoopGroup 설정
         .channel(NioSocketChannel.class)           // Channel 클래스 지정
         .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)  // 옵션 설정
         .handler(new ChannelInitializer<SocketChannel>() {   // Handler 설정
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline().addLast(new YourHandler());
             }
         });

ChannelFuture future = bootstrap.connect("example.com", 80);
```

**클라이언트 개발 시 적용**:
- `option()`으로 타임아웃, KeepAlive 등 설정
- `handler()`로 ChannelPipeline 초기화
- `connect()` 대신 Connection Pool 사용 권장

#### 섹션 4.3: EventLoop (스레드 모델)

**핵심 개념**:
- 하나의 EventLoop = 하나의 Thread
- 각 Channel은 하나의 EventLoop에 바인딩
- EventLoop는 여러 Channel을 처리

**클라이언트 개발 시 적용**:
```java
// Worker 스레드 수 결정
int threads = Runtime.getRuntime().availableProcessors();  // 기본값
EventLoopGroup group = new MultiThreadIoEventLoopGroup(threads, NioIoHandler.newFactory());
```

**권장 스레드 수**:
- CPU 바운드: CPU 코어 수
- I/O 바운드: CPU 코어 수 * 2

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 4.3.1 (새로운 아키텍처)

#### 섹션 4.5: ChannelPipeline (Handler 체인)

**핵심 개념**:
```
[Inbound]  네트워크 → Handler1 → Handler2 → Handler3 → 애플리케이션
[Outbound] 애플리케이션 → Handler3 → Handler2 → Handler1 → 네트워크
```

**클라이언트 개발 시 적용**:
```java
ChannelPipeline p = ch.pipeline();

// Inbound (응답 처리 순서)
p.addLast("readTimeout", new ReadTimeoutHandler(30));
p.addLast("decoder", new HttpResponseDecoder());
p.addLast("aggregator", new HttpObjectAggregator(8192));
p.addLast("handler", new MyResponseHandler());

// Outbound (요청 전송 순서는 자동으로 역순)
p.addLast("encoder", new HttpRequestEncoder());
```

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 4.5

#### 섹션 5: ByteBuf (메모리 관리)

**핵심 원칙**:
- **Reference Counting**: `retain()` / `release()`
- **메모리 누수 방지**: 사용 후 반드시 `release()`

```java
ByteBuf buf = Unpooled.buffer(256);
try {
    buf.writeBytes("Hello".getBytes());
    // ... 사용
} finally {
    buf.release();  // 필수!
}
```

**클라이언트 개발 시 적용**:
- `SimpleChannelInboundHandler` 사용 시 자동 release
- 직접 `ChannelInboundHandlerAdapter` 사용 시 수동 release 필요

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 5.1

#### 섹션 6: Codec (프로토콜 변환)

**핵심 Codec**:
- `HttpRequestEncoder`: HTTP 요청 → 바이트
- `HttpResponseDecoder`: 바이트 → HTTP 응답
- `HttpObjectAggregator`: Chunked 메시지 합치기

```java
p.addLast("decoder", new HttpResponseDecoder());
p.addLast("aggregator", new HttpObjectAggregator(10 * 1024 * 1024)); // 10MB
p.addLast("encoder", new HttpRequestEncoder());
```

**클라이언트 개발 시 적용**:
- HTTP 클라이언트: 기본 Codec 사용
- 커스텀 프로토콜: `ByteToMessageDecoder`, `MessageToByteEncoder` 상속

> 📖 **상세 내용**: `NETTY_분석_가이드.md` 섹션 6

---

## 7. 체크리스트 및 권장사항

### 7.1 구현 체크리스트

#### 필수 기능

- [ ] **Bootstrap 설정**
  - [ ] EventLoopGroup 생성 및 설정
  - [ ] Channel 클래스 선택 (NioSocketChannel)
  - [ ] ChannelOption 설정 (타임아웃, KeepAlive)

- [ ] **ChannelPipeline 구성**
  - [ ] HTTP Codec 추가 (Encoder/Decoder)
  - [ ] HttpObjectAggregator 추가
  - [ ] Timeout Handler 추가 (Read/Write)
  - [ ] 커스텀 Response Handler 추가

- [ ] **비동기 API**
  - [ ] CompletableFuture 또는 Reactive 방식 지원
  - [ ] 동기 API는 비동기 위에 래퍼로 제공

- [ ] **리소스 관리**
  - [ ] AutoCloseable 구현
  - [ ] EventLoopGroup 종료 (shutdownGracefully)
  - [ ] ByteBuf 해제 (Reference Counting)

- [ ] **에러 처리**
  - [ ] ChannelHandler에서 exceptionCaught 구현
  - [ ] Timeout 에러 처리
  - [ ] 네트워크 에러 처리

#### 권장 기능

- [ ] **Connection Pool**
  - [ ] ChannelPool 사용
  - [ ] 최대 연결 수 제한
  - [ ] Idle 연결 정리

- [ ] **재시도 로직**
  - [ ] 재시도 가능 에러 판단
  - [ ] 지수 백오프 (Exponential Backoff)
  - [ ] 최대 재시도 횟수 제한

- [ ] **메트릭 및 모니터링**
  - [ ] 연결 풀 메트릭 (활성/유휴 연결 수)
  - [ ] 요청 메트릭 (성공/실패 횟수)
  - [ ] 응답 시간 측정
  - [ ] 데이터 전송량 측정

- [ ] **로깅**
  - [ ] 요청/응답 로깅
  - [ ] 에러 로깅
  - [ ] Wire 로깅 (디버그용)

### 7.2 성능 최적화 권장사항

#### EventLoopGroup 크기

```java
// CPU 바운드 작업
int threads = Runtime.getRuntime().availableProcessors();

// I/O 바운드 작업 (권장)
int threads = Runtime.getRuntime().availableProcessors() * 2;

EventLoopGroup group = new MultiThreadIoEventLoopGroup(threads, NioIoHandler.newFactory());
```

#### ChannelOption 튜닝

```java
bootstrap
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
    .option(ChannelOption.SO_KEEPALIVE, true)        // TCP KeepAlive
    .option(ChannelOption.TCP_NODELAY, true)         // Nagle 알고리즘 비활성화
    .option(ChannelOption.SO_REUSEADDR, true)        // TIME_WAIT 재사용
    .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT); // 풀링된 ByteBuf
```

#### Connection Pool 크기

```
최대 연결 수 = (예상 동시 요청 수) * 1.2
```

예: 초당 1000 요청, 평균 응답 시간 50ms
```
동시 요청 수 = 1000 * 0.05 = 50
최대 연결 수 = 50 * 1.2 = 60
```

#### 메모리 최적화

```java
// HttpObjectAggregator 크기 제한
new HttpObjectAggregator(10 * 1024 * 1024)  // 10MB

// Direct Buffer 사용 (off-heap)
bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

### 7.3 보안 권장사항

#### TLS/SSL 설정

```java
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;

SslContext sslContext = SslContextBuilder.forClient()
    .trustManager(InsecureTrustManagerFactory.INSTANCE)  // 개발용
    .build();

pipeline.addFirst("ssl", sslContext.newHandler(ch.alloc(), host, port));
```

**프로덕션 환경**:
- 신뢰할 수 있는 TrustManager 사용
- 인증서 검증 활성화
- TLS 1.2 이상 사용

#### Timeout 설정

```java
// 연결 타임아웃
bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);

// 읽기 타임아웃
pipeline.addLast("readTimeout", new ReadTimeoutHandler(30, TimeUnit.SECONDS));

// 쓰기 타임아웃
pipeline.addLast("writeTimeout", new WriteTimeoutHandler(30, TimeUnit.SECONDS));
```

### 7.4 테스트 권장사항

#### 단위 테스트

```java
@Test
public void testHttpRequest() {
    EmbeddedChannel channel = new EmbeddedChannel(
        new HttpResponseDecoder(),
        new HttpObjectAggregator(8192),
        new MyResponseHandler()
    );

    // 가짜 HTTP 응답 주입
    String response = "HTTP/1.1 200 OK\r\n" +
                      "Content-Length: 5\r\n" +
                      "\r\n" +
                      "Hello";

    channel.writeInbound(Unpooled.copiedBuffer(response.getBytes()));

    // 결과 검증
    FullHttpResponse resp = channel.readInbound();
    assertEquals(200, resp.status().code());
}
```

#### 통합 테스트

```java
@Test
public void testRealHttpRequest() {
    SimpleHttpClient client = SimpleHttpClient.builder()
        .connectTimeout(Duration.ofSeconds(5))
        .build();

    try {
        HttpResponse response = client.get("http://example.com");
        assertEquals(200, response.getStatusCode());
    } finally {
        client.close();
    }
}
```

#### 부하 테스트

- **도구**: JMeter, Gatling, wrk
- **메트릭**: TPS, 응답 시간, 에러율
- **시나리오**: 정상, 피크, 장시간

---

## 8. 참고 자료

### 8.1 공식 문서

- [Netty User Guide](https://netty.io/wiki/user-guide-for-4.x.html)
- [Netty API Documentation](https://netty.io/4.1/api/index.html)
- [Reactor Netty Reference](https://docs.spring.io/projectreactor/reactor-netty/docs/current/reference/html/)
- [Ktor Documentation](https://ktor.io/docs/server-engines.html)

### 8.2 오픈소스 참고 코드

#### Spring WebFlux (Reactor Netty)
- **GitHub**: [reactor/reactor-netty](https://github.com/reactor/reactor-netty)
- **핵심 클래스**:
  - `reactor.netty.http.client.HttpClient`
  - `reactor.netty.resources.ConnectionProvider`
  - `reactor.netty.resources.LoopResources`

#### Ktor
- **GitHub**: [ktorio/ktor](https://github.com/ktorio/ktor)
- **핵심 파일**:
  - `ktor-server/ktor-server-netty/jvm/src/io/ktor/server/netty/NettyApplicationEngine.kt`
  - `ktor-server/ktor-server-netty/jvm/src/io/ktor/server/netty/EngineMain.kt`

#### AWS SDK for Java 2.x
- **GitHub**: [aws/aws-sdk-java-v2](https://github.com/aws/aws-sdk-java-v2)
- **Netty HTTP Client**: `NettyNioAsyncHttpClient`

### 8.3 추가 학습 자료

#### 아티클
- [Spring WebFlux — Under the hood](https://medium.com/@diego.lucasilva/spring-webflux-under-the-hood-c6446c87ea84)
- [Spring WebFlux Internals: How Netty's Event Loop & Threads Power Reactive Apps](https://medium.com/@gourav20056/spring-webflux-internals-how-nettys-event-loop-threads-power-reactive-apps-4698c144ef68)
- [Introduction to Netty | Baeldung](https://www.baeldung.com/netty)
- [Building a simple Netty server and client](https://medium.com/@cjz.lxg/building-a-simple-netty-server-and-client-d95061156313)

#### 튜토리얼
- [A Quick Guide to Java on Netty | Okta Developer](https://developer.okta.com/blog/2019/11/25/java-netty-webflux)
- [Spring Boot Reactor Netty Configuration | Baeldung](https://www.baeldung.com/spring-boot-reactor-netty)

### 8.4 이 프로젝트의 문서

- **NETTY_분석_가이드.md**: Netty 핵심 개념 상세 설명
  - 섹션 1-4: 기초 개념
  - 섹션 5-6: ByteBuf, Codec
  - 섹션 7: Echo/HTTP 서버 예제
  - 섹션 11: Netty 4.2 마이그레이션 가이드

---

## 9. 마치며

이 가이드는 **실제 프로덕션 환경에서 검증된 Spring WebFlux와 Ktor의 Netty 활용 방식**을 분석하여, 클라이언트 라이브러리 개발에 필요한 핵심 패턴과 Best Practice를 정리했습니다.

### 핵심 요약

1. **리소스 라이프사이클 관리**: EventLoopGroup, Bootstrap을 재사용하고 종료 시 정리
2. **Connection Pool 구현**: 연결 재사용으로 성능 향상
3. **비동기 API 우선**: CompletableFuture 또는 Reactive Streams 사용
4. **Timeout 계층화**: Connect, Read, Write, Request 각각 설정
5. **메트릭 수집**: 성능 모니터링 및 문제 진단

### 다음 단계

- [ ] [5.1 기본 클라이언트 구현](#51-기본-클라이언트-구현) 코드를 직접 작성
- [ ] Connection Pool 추가
- [ ] 재시도 로직 구현
- [ ] 메트릭 수집 추가
- [ ] TLS/SSL 지원 추가
- [ ] `NETTY_분석_가이드.md`의 고급 주제 학습

**Happy Coding!** 🚀
