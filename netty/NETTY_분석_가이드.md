# Netty 프로젝트 분석 가이드

> **목표**: Netty 오픈소스 프로젝트를 처음 접하는 개발자가 체계적으로 코드를 이해할 수 있도록 안내

---

## 📋 목차

1. [Netty 개요](#1-netty-개요)
2. [프로젝트 구조](#2-프로젝트-구조)
3. [분석 순서 로드맵](#3-분석-순서-로드맵)
4. [Phase 1: 핵심 아키텍처](#4-phase-1-핵심-아키텍처)
5. [Phase 2: 메모리 관리](#5-phase-2-메모리-관리)
6. [Phase 3: 프로토콜 코덱](#6-phase-3-프로토콜-코덱)
7. [Phase 4: 실전 예제 분석](#7-phase-4-실전-예제-분석)
8. [추가 학습 자료](#8-추가-학습-자료)
9. [분석 체크리스트 (전체)](#9-분석-체크리스트-전체)
10. [다음 단계](#10-다음-단계)
11. [Deprecated API 마이그레이션 가이드](#11-deprecated-api-마이그레이션-가이드) ⭐ NEW

---

## 1. Netty 개요

**Netty**는 비동기 이벤트 기반 네트워크 애플리케이션 프레임워크입니다.

### 핵심 특징
- **비동기 Non-blocking I/O**: Java NIO 기반
- **이벤트 기반 아키텍처**: Reactor 패턴 구현
- **높은 성능**: Zero-copy, 메모리 풀링
- **다양한 프로토콜 지원**: HTTP, WebSocket, gRPC 등

### 사용 사례
- 고성능 서버/클라이언트
- 게임 서버
- 메시징 시스템
- RPC 프레임워크 (gRPC)

---

## 2. 프로젝트 구조

```
netty/
├── buffer/              ← 메모리 관리 (ByteBuf, ByteBufAllocator)
├── codec/              ← 기본 코덱 (Encoder, Decoder)
├── codec-http/         ← HTTP/1.x, WebSocket
├── codec-http2/        ← HTTP/2
├── codec-http3/        ← HTTP/3 (QUIC)
├── codec-mqtt/         ← MQTT (IoT)
├── codec-redis/        ← Redis 프로토콜
├── common/             ← 공통 유틸리티
├── handler/            ← SSL, 타임아웃 등
├── transport/          ← 핵심 전송 계층
│   ├── Channel, EventLoop, Pipeline
│   └── Bootstrap
└── transport-native-*  ← 네이티브 최적화 (epoll, kqueue)
```

---

## 3. 분석 순서 로드맵

```
Phase 1: 핵심 아키텍처 이해 (필수)
   ↓
Phase 2: 메모리 관리 시스템
   ↓
Phase 3: 프로토콜 코덱 구조
   ↓
Phase 4: 실전 예제 분석
```

**권장 학습 시간**: 각 Phase 당 2-4시간

---

## 4. Phase 1: 핵심 아키텍처

> **목표**: Netty의 기본 동작 원리와 핵심 컴포넌트 이해

### 4.1 시작점: Bootstrap

**학습 순서**:
1. `ServerBootstrap` - 서버 시작
2. `Bootstrap` - 클라이언트 시작

#### 파일 위치
```
transport/src/main/java/io/netty/bootstrap/
├── AbstractBootstrap.java      (공통 부모 클래스)
├── Bootstrap.java               (클라이언트)
└── ServerBootstrap.java         (서버)
```

#### AbstractBootstrap (추상 클래스)
**역할**: 클라이언트/서버 부트스트랩의 공통 기능 제공

**주요 필드**:
```java
abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> {
    volatile EventLoopGroup group;           // EventLoop 그룹
    volatile ChannelFactory<? extends C> channelFactory;  // Channel 팩토리
    private final Map<ChannelOption<?>, Object> options;  // 옵션 설정
    private final Map<AttributeKey<?>, Object> attrs;     // Attribute 설정
    volatile ChannelHandler handler;         // 초기 핸들러
}
```

**핵심 메서드**:
- `group(EventLoopGroup)`: EventLoop 할당
- `channel(Class)`: Channel 타입 지정
- `handler(ChannelHandler)`: 핸들러 설정
- `option(ChannelOption, Object)`: 옵션 설정

#### ServerBootstrap (구체 클래스)
**역할**: 서버 채널 시작 및 관리

**추가 필드**:
```java
public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {
    volatile EventLoopGroup childGroup;      // 자식 Channel용 EventLoop
    volatile ChannelHandler childHandler;    // 자식 Channel 핸들러
    private final Map<ChannelOption<?>, Object> childOptions;
    private final Map<AttributeKey<?>, Object> childAttrs;
}
```

**핵심 메서드**:
- `group(EventLoopGroup parent, EventLoopGroup child)`: 부모/자식 EventLoop 분리
- `childHandler(ChannelHandler)`: 클라이언트 연결용 핸들러
- `bind(int port)`: 포트 바인딩 → ChannelFuture 반환

#### Bootstrap (구체 클래스)
**역할**: 클라이언트 채널 시작

**추가 필드**:
```java
public class Bootstrap extends AbstractBootstrap<Bootstrap, Channel> {
    volatile AddressResolverGroup<?> resolver;  // 주소 해석
    volatile SocketAddress remoteAddress;       // 원격 주소
}
```

**핵심 메서드**:
- `connect(String host, int port)`: 서버 연결 → ChannelFuture 반환
- `remoteAddress(SocketAddress)`: 원격 주소 설정

#### 사용 예제 (권장 방식)

```java
// 서버 (최신 권장 방식)
EventLoopGroup bossGroup = new MultiThreadIoEventLoopGroup(1, NioIoHandler.newFactory());
EventLoopGroup workerGroup = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .handler(new LoggingHandler(LogLevel.INFO))
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) {
             ch.pipeline().addLast(new EchoServerHandler());
         }
     });

    ChannelFuture f = b.bind(8080).sync();
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

#### 사용 예제 (레거시 방식)

> ⚠️ **Deprecated 경고**: `NioEventLoopGroup`은 Netty 4.2에서 deprecated 되었습니다.
> 새 프로젝트에서는 `MultiThreadIoEventLoopGroup + NioIoHandler.newFactory()` 사용을 권장합니다.

```java
// 서버 (레거시 방식 - 학습 목적으로 제공)
EventLoopGroup bossGroup = new NioEventLoopGroup(1);  // Deprecated
EventLoopGroup workerGroup = new NioEventLoopGroup(); // Deprecated
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .handler(new LoggingHandler(LogLevel.INFO))
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) {
             ch.pipeline().addLast(new EchoServerHandler());
         }
     });

    ChannelFuture f = b.bind(8080).sync();
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

#### 두 방식 비교

| 항목 | NioEventLoopGroup (Old) | MultiThreadIoEventLoopGroup (New) |
|------|-------------------------|-----------------------------------|
| **상태** | Deprecated | ✅ 권장 |
| **Transport 결합도** | NIO에 강하게 결합 | IoHandler로 추상화 |
| **확장성** | Transport별 별도 클래스 필요 | IoHandlerFactory로 통합 |
| **코드 중복** | 높음 (Epoll, KQueue 각각 구현) | 낮음 (공통 로직 공유) |
| **호환성** | EventLoopGroup 인터페이스 | EventLoopGroup 인터페이스 (동일) |
| **성능** | 동일 | 동일 |

**마이그레이션 참고사항**:
- 두 방식 모두 `EventLoopGroup` 인터페이스를 구현하므로 API 호환성 유지
- `NioEventLoopGroup`은 내부적으로 이미 새 아키텍처를 사용하므로 성능 차이 없음
- 편한 시점에 마이그레이션 가능 (급하지 않음)

### 4.2 Channel - 네트워크 연결 추상화

#### 파일 위치
```
transport/src/main/java/io/netty/channel/
├── Channel.java                 (인터페이스)
├── AbstractChannel.java         (추상 구현)
└── socket/
    ├── ServerSocketChannel.java (서버 소켓 인터페이스)
    ├── SocketChannel.java       (클라이언트 소켓 인터페이스)
    └── nio/
        ├── NioServerSocketChannel.java  (NIO 서버 구현)
        └── NioSocketChannel.java        (NIO 클라이언트 구현)
```

#### Channel (인터페이스)
**역할**: 네트워크 I/O 작업의 추상화

**핵심 메서드**:
```java
public interface Channel extends AttributeMap, ChannelOutboundInvoker, Comparable<Channel> {
    ChannelId id();                          // 고유 식별자
    EventLoop eventLoop();                   // 할당된 EventLoop
    Channel parent();                        // 부모 Channel (ServerSocket의 경우)
    ChannelConfig config();                  // 설정
    boolean isOpen();                        // 열림 상태
    boolean isRegistered();                  // EventLoop 등록 여부
    boolean isActive();                      // 활성화 여부 (bind/connect 완료)
    ChannelMetadata metadata();              // 메타데이터
    SocketAddress localAddress();            // 로컬 주소
    SocketAddress remoteAddress();           // 원격 주소
    ChannelFuture closeFuture();             // 종료 Future
    boolean isWritable();                    // 쓰기 가능 여부

    Unsafe unsafe();                         // 내부 작업용
    ChannelPipeline pipeline();              // 파이프라인
    ByteBufAllocator alloc();                // 메모리 할당자

    // Outbound 작업
    ChannelFuture bind(SocketAddress localAddress);
    ChannelFuture connect(SocketAddress remoteAddress);
    ChannelFuture disconnect();
    ChannelFuture close();
    ChannelFuture deregister();
    ChannelFuture write(Object msg);
    ChannelFuture writeAndFlush(Object msg);
    Channel read();
    Channel flush();
}
```

#### AbstractChannel (추상 클래스)
**역할**: Channel의 기본 구현

**주요 필드**:
```java
public abstract class AbstractChannel implements Channel {
    private final Channel parent;                    // 부모 Channel
    private final ChannelId id;                      // 고유 ID
    private final DefaultChannelPipeline pipeline;   // 파이프라인
    private final VoidChannelPromise unsafeVoidPromise;

    private volatile SocketAddress localAddress;
    private volatile SocketAddress remoteAddress;
    private volatile EventLoop eventLoop;
    private volatile boolean registered;

    protected abstract class AbstractUnsafe implements Unsafe {
        // 실제 I/O 작업 수행
        public final void register(EventLoop eventLoop, ChannelPromise promise);
        public final void bind(SocketAddress localAddress, ChannelPromise promise);
        public final void connect(SocketAddress remoteAddress, ...);
        public final void write(Object msg, ChannelPromise promise);
        public final void flush();
        public final void close(ChannelPromise promise);
    }
}
```

#### NioServerSocketChannel (구체 클래스)
**역할**: NIO 기반 서버 소켓 구현

**주요 필드**:
```java
public class NioServerSocketChannel extends AbstractNioMessageChannel
                                     implements io.netty.channel.socket.ServerSocketChannel {
    private final ServerSocketChannelConfig config;  // 설정

    public NioServerSocketChannel() {
        this(DEFAULT_SELECTOR_PROVIDER);  // SelectorProvider.provider()
    }

    public NioServerSocketChannel(SelectorProvider provider) {
        this(provider, null);
    }

    public NioServerSocketChannel(SelectorProvider provider, InternetProtocolFamily family) {
        this(newChannel(provider, family));  // Java NIO ServerSocketChannel 생성
    }

    private static ServerSocketChannel newChannel(SelectorProvider provider, InternetProtocolFamily family) {
        return provider.openServerSocketChannel(toProtocolFamily(family));
    }

    @Override
    protected int doReadMessages(List<Object> buf) throws Exception {
        // 새 연결 수락
        SocketChannel ch = SocketUtils.accept(javaChannel());
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));  // 자식 Channel 생성
            return 1;
        }
        return 0;
    }
}
```

#### Channel 상태 전이 다이어그램

```
┌──────────────────────────────────────────────────────────┐
│                     Channel Lifecycle                     │
└──────────────────────────────────────────────────────────┘

    CREATED
       ↓
  channelRegistered() ─────→ EventLoop에 등록
       ↓
  channelActive() ──────────→ bind()/connect() 성공
       ↓
  [데이터 송수신]
       ↓
  channelInactive() ────────→ close() 완료
       ↓
  channelUnregistered() ────→ EventLoop에서 제거

상태 플래그:
- isOpen(): Channel 생성 ~ close() 완료
- isRegistered(): EventLoop 등록 ~ 제거
- isActive(): bind/connect 성공 ~ close 시작
```

### 4.3 EventLoop & EventLoopGroup - 이벤트 처리 엔진

#### 4.3.1 새로운 아키텍처 (권장)

> ✅ **Netty 4.2 권장 방식**: 이 섹션은 최신 아키텍처를 설명합니다.

##### 파일 위치
```
transport/src/main/java/io/netty/channel/
├── IoEventLoop.java                    (인터페이스)
├── IoEventLoopGroup.java               (인터페이스)
├── MultiThreadIoEventLoopGroup.java    (구현)
├── SingleThreadIoEventLoop.java        (구현)
├── IoHandler.java                      (인터페이스)
├── IoHandlerFactory.java               (인터페이스)
└── nio/
    └── NioIoHandler.java               (NIO 구현)
```

##### 핵심 개념

**IoEventLoopGroup (인터페이스)**
- 역할: EventLoop 관리 및 IoHandle(Channel 포함) 등록
- 특징: Transport 독립적인 설계

**MultiThreadIoEventLoopGroup (구현 클래스)**
- 역할: 다중 스레드 EventLoop 관리
- 특징: IoHandlerFactory를 통해 다양한 Transport 지원

**IoHandler (인터페이스)**
- 역할: I/O 작업 추상화 (Selector 관리, I/O 처리)
- 구현체: NioIoHandler, EpollIoHandler, KQueueIoHandler, IoUringIoHandler

**IoHandlerFactory (인터페이스)**
- 역할: Transport별 IoHandler 생성 팩토리
- 사용법: `NioIoHandler.newFactory()`, `EpollIoHandler.newFactory()`

##### 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────┐
│              새로운 IoEventLoopGroup 아키텍처                │
└─────────────────────────────────────────────────────────────┘

IoEventLoopGroup (인터페이스)
    ↑
MultiThreadIoEventLoopGroup (구현)
    │
    ├─── IoHandlerFactory ──→ NioIoHandler.newFactory()
    │                          EpollIoHandler.newFactory()
    │                          KQueueIoHandler.newFactory()
    │                          IoUringIoHandler.newFactory()
    │
    └─── IoEventLoop[] (SingleThreadIoEventLoop)
            │
            └─── IoHandler (NioIoHandler, EpollIoHandler, ...)
                    ├─── Selector (NIO의 경우)
                    └─── I/O 처리 로직

장점:
1. Transport 추상화: NIO, Epoll, KQueue 등을 동일한 API로 사용
2. 코드 중복 제거: 공통 로직을 MultiThreadIoEventLoopGroup에서 관리
3. 확장성: 커스텀 IoHandler 구현 가능 (예: io_uring)
4. 범용성: Channel 외 IoHandle로 일반화 (File, Socket 등)
```

##### 사용 예제

**기본 사용법**:
```java
// NIO Transport (가장 일반적)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());

// 스레드 수 지정
EventLoopGroup bossGroup = new MultiThreadIoEventLoopGroup(1, NioIoHandler.newFactory());
EventLoopGroup workerGroup = new MultiThreadIoEventLoopGroup(4, NioIoHandler.newFactory());
```

**Transport별 사용법**:
```java
// Epoll Transport (Linux 고성능)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(EpollIoHandler.newFactory());

// KQueue Transport (macOS/BSD)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(KQueueIoHandler.newFactory());

// io_uring Transport (Linux 최신 고성능)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(IoUringIoHandler.newFactory());
```

**고급 설정**:
```java
// ThreadFactory 커스터마이징
ThreadFactory threadFactory = new DefaultThreadFactory("netty-nio");
EventLoopGroup group = new MultiThreadIoEventLoopGroup(
    8,  // 스레드 수
    threadFactory,
    NioIoHandler.newFactory()
);

// SelectorProvider 커스터마이징
SelectorProvider provider = SelectorProvider.provider();
SelectStrategyFactory strategy = DefaultSelectStrategyFactory.INSTANCE;
EventLoopGroup group = new MultiThreadIoEventLoopGroup(
    8,
    NioIoHandler.newFactory(provider, strategy)
);
```

##### IoHandler 내부 구조

**NioIoHandler 역할**:
```java
public final class NioIoHandler implements IoHandler {
    // Selector 관리
    private Selector selector;
    private final SelectorProvider provider;
    private final SelectStrategy selectStrategy;

    // IoHandle 등록 (Channel 등)
    @Override
    public IoRegistration register(IoHandle handle) throws Exception {
        NioIoHandle nioHandle = (NioIoHandle) handle;
        SelectionKey key = nioHandle.selectableChannel()
            .register(selector, ops, attachment);
        return new DefaultNioRegistration(key);
    }

    // I/O 이벤트 처리
    @Override
    public int run(IoHandlerContext context) {
        // 1. select() I/O 대기
        // 2. I/O 이벤트 처리 (OP_ACCEPT, OP_READ, OP_WRITE)
        // 3. 완료된 작업 수 반환
        return processSelectedKeys();
    }
}
```

**변경 이유**:
1. **코드 중복 제거**: Transport별로 NioEventLoop, EpollEventLoop 등 각각 구현 필요 없음
2. **확장성 향상**: 새로운 Transport 추가 시 IoHandler만 구현하면 됨
3. **일반화**: Channel뿐 아니라 File, Pipe 등도 IoHandle로 등록 가능

#### 4.3.2 기존 아키텍처 (Deprecated)

> ⚠️ **Deprecated 경고**: 다음 내용은 레거시 방식입니다.
> 학습 목적으로 제공하며, 실제 코드에서는 4.3.1의 새 방식을 사용하세요.

##### 파일 위치 (레거시)
```
transport/src/main/java/io/netty/channel/
├── EventLoop.java                  (인터페이스)
├── EventLoopGroup.java             (인터페이스)
├── MultithreadEventLoopGroup.java  (추상 구현)
└── nio/
    ├── NioEventLoop.java           (NIO 구현) - Deprecated
    └── NioEventLoopGroup.java      (NIO 그룹) - Deprecated
```

#### EventLoop (인터페이스)
**역할**: 단일 스레드 이벤트 루프

**상속 관계**:
```java
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {
    @Override
    EventLoopGroup parent();  // 소속 그룹
}

// 계층 구조
EventExecutorGroup
    ↑
EventLoopGroup
    ↑
EventLoop
    ↑
OrderedEventExecutor (순차 실행 보장)
```

#### EventLoopGroup (인터페이스)
**역할**: EventLoop 관리 및 Channel 등록

**핵심 메서드**:
```java
public interface EventLoopGroup extends EventExecutorGroup {
    @Override
    EventLoop next();  // 다음 EventLoop 선택

    ChannelFuture register(Channel channel);  // Channel 등록
    ChannelFuture register(ChannelPromise promise);
}
```

#### MultithreadEventLoopGroup (추상 클래스)
**역할**: 다중 스레드 EventLoop 관리

**주요 필드**:
```java
public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup
                                                 implements EventLoopGroup {
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }

    private static final int DEFAULT_EVENT_LOOP_THREADS =
        Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads",
            NettyRuntime.availableProcessors() * 2));  // CPU 코어 * 2
}
```

#### NioEventLoop (구체 클래스)
**역할**: Java NIO Selector 기반 이벤트 루프

**주요 필드**:
```java
public final class NioEventLoop extends SingleThreadEventLoop {
    private final Selector selector;              // Java NIO Selector
    private final SelectorProvider provider;

    private final SelectStrategy selectStrategy;
    private final SelectedSelectionKeySet selectedKeys;

    @Override
    protected void run() {
        int selectCnt = 0;
        for (;;) {
            try {
                int strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.BUSY_WAIT:
                case SelectStrategy.SELECT:
                    // I/O 이벤트 대기
                    select(wakenUp.getAndSet(false));

                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                }

                // I/O 작업 처리 비율 (기본 50%)
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                if (ioRatio == 100) {
                    processSelectedKeys();  // I/O 이벤트 처리
                    ranTasks = runAllTasks();  // 모든 작업 실행
                } else {
                    long ioStartTime = System.nanoTime();
                    processSelectedKeys();  // I/O 이벤트 처리
                    long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);  // 시간 제한 작업 실행
                }
            } catch (CancelledKeyException e) {
                // 처리
            }
        }
    }
}
```

#### EventLoop 처리 흐름 다이어그램

```
┌─────────────────────────────────────────────────────────────┐
│                    EventLoop Run Loop                        │
└─────────────────────────────────────────────────────────────┘

  ┌──────────────────────────┐
  │  1. select() I/O 대기    │
  │  (timeout 고려)          │
  └───────────┬──────────────┘
              │
  ┌───────────▼──────────────┐
  │  2. I/O 이벤트 처리      │
  │  - OP_ACCEPT             │
  │  - OP_CONNECT            │
  │  - OP_READ               │
  │  - OP_WRITE              │
  └───────────┬──────────────┘
              │
  ┌───────────▼──────────────┐
  │  3. Task Queue 처리      │
  │  - 사용자 제출 작업      │
  │  - 스케줄된 작업         │
  └───────────┬──────────────┘
              │
              └──────────────→ (반복)

ioRatio 조절:
- ioRatio = 50 (기본): I/O 50%, Task 50%
- ioRatio = 70: I/O 70%, Task 30%
- ioRatio = 100: I/O 우선, 모든 Task 실행
```

#### EventLoopGroup 할당 전략

```
ServerBootstrap 시나리오:
┌────────────────────────────────────────────────────┐
│  bossGroup (1 스레드)                              │
│  ├─ EventLoop-1                                    │
│      └─ NioServerSocketChannel (포트 8080)         │
│         역할: 새 연결 수락 (OP_ACCEPT)             │
└────────────────────────────────────────────────────┘
                     │
                     │ accept() → NioSocketChannel 생성
                     ↓
┌────────────────────────────────────────────────────┐
│  workerGroup (N 스레드, 기본 CPU * 2)              │
│  ├─ EventLoop-1                                    │
│  │   ├─ NioSocketChannel (Client A)                │
│  │   └─ NioSocketChannel (Client B)                │
│  ├─ EventLoop-2                                    │
│  │   └─ NioSocketChannel (Client C)                │
│  └─ EventLoop-N                                    │
│      └─ NioSocketChannel (Client D)                │
│         역할: 읽기/쓰기 (OP_READ, OP_WRITE)         │
└────────────────────────────────────────────────────┘

라운드 로빈 할당:
- 새 Channel이 등록될 때마다 next() 호출
- 부하 분산
```

#### 4.3.3 마이그레이션 가이드

##### 기존 코드에서 새 코드로 전환

**패턴 1: 기본 사용**
```java
// Before (Deprecated)
EventLoopGroup group = new NioEventLoopGroup();

// After (Recommended)
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
```

**패턴 2: 스레드 수 지정**
```java
// Before
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(4);

// After
EventLoopGroup bossGroup = new MultiThreadIoEventLoopGroup(1, NioIoHandler.newFactory());
EventLoopGroup workerGroup = new MultiThreadIoEventLoopGroup(4, NioIoHandler.newFactory());
```

**패턴 3: Epoll 사용 (Linux)**
```java
// Before
EventLoopGroup group = new EpollEventLoopGroup();

// After
EventLoopGroup group = new MultiThreadIoEventLoopGroup(EpollIoHandler.newFactory());
```

**패턴 4: KQueue 사용 (macOS/BSD)**
```java
// Before
EventLoopGroup group = new KQueueEventLoopGroup();

// After
EventLoopGroup group = new MultiThreadIoEventLoopGroup(KQueueIoHandler.newFactory());
```

**패턴 5: ThreadFactory 커스터마이징**
```java
// Before
ThreadFactory threadFactory = new DefaultThreadFactory("netty-nio");
EventLoopGroup group = new NioEventLoopGroup(8, threadFactory);

// After
ThreadFactory threadFactory = new DefaultThreadFactory("netty-nio");
EventLoopGroup group = new MultiThreadIoEventLoopGroup(8, threadFactory, NioIoHandler.newFactory());
```

**패턴 6: 고급 설정**
```java
// Before
NioEventLoopGroup group = new NioEventLoopGroup(
    8,
    threadFactory,
    SelectorProvider.provider(),
    DefaultSelectStrategyFactory.INSTANCE
);

// After
MultiThreadIoEventLoopGroup group = new MultiThreadIoEventLoopGroup(
    8,
    threadFactory,
    NioIoHandler.newFactory(
        SelectorProvider.provider(),
        DefaultSelectStrategyFactory.INSTANCE
    )
);
```

##### 호환성 참고사항

**API 호환성**:
```
✅ 두 방식 모두 EventLoopGroup 인터페이스 사용하므로 대부분 호환
✅ ServerBootstrap, Bootstrap API 변경 없음
✅ ChannelPipeline, Handler 코드 변경 불필요
```

**내부 동작**:
- `NioEventLoopGroup`은 내부적으로 이미 새 아키텍처를 사용 중
- 단순한 래퍼(wrapper) 역할만 수행
- 성능 차이 없음

**마이그레이션 시기**:
- 기존 코드는 계속 작동 (제거되지 않음)
- 편한 시점에 전환 가능 (급하지 않음)
- 새 프로젝트는 처음부터 새 방식 권장

**주의사항**:
1. Import 문 변경 필요:
   ```java
   // Before
   import io.netty.channel.nio.NioEventLoopGroup;

   // After
   import io.netty.channel.MultiThreadIoEventLoopGroup;
   import io.netty.channel.nio.NioIoHandler;
   ```

2. 특정 메서드는 제거됨:
   ```java
   // NioEventLoopGroup.setIoRatio() - deprecated, no-op
   // 새 아키텍처에서는 IoHandler가 처리
   ```

##### 마이그레이션 체크리스트

- [ ] NioEventLoopGroup → MultiThreadIoEventLoopGroup + NioIoHandler.newFactory()
- [ ] EpollEventLoopGroup → MultiThreadIoEventLoopGroup + EpollIoHandler.newFactory()
- [ ] KQueueEventLoopGroup → MultiThreadIoEventLoopGroup + KQueueIoHandler.newFactory()
- [ ] Import 문 업데이트
- [ ] 테스트 코드 실행
- [ ] 통합 테스트 확인

##### 자동 변환 참고

```bash
# NioEventLoopGroup 사용처 찾기
grep -r "new NioEventLoopGroup" --include="*.java"

# 수동 변환 예시
# Before: new NioEventLoopGroup()
# After:  new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory())

# Before: new NioEventLoopGroup(1)
# After:  new MultiThreadIoEventLoopGroup(1, NioIoHandler.newFactory())
```

### 4.4 ChannelPipeline & ChannelHandler - 이벤트 처리 체인

#### 파일 위치
```
transport/src/main/java/io/netty/channel/
├── ChannelPipeline.java          (인터페이스)
├── DefaultChannelPipeline.java   (구현)
├── ChannelHandler.java           (인터페이스)
├── ChannelHandlerContext.java    (인터페이스)
├── ChannelInboundHandler.java    (인터페이스)
├── ChannelOutboundHandler.java   (인터페이스)
└── ChannelInitializer.java       (추상 클래스)
```

#### ChannelPipeline (인터페이스)
**역할**: Handler 체인 관리 (Intercepting Filter 패턴)

**핵심 메서드**:
```java
public interface ChannelPipeline extends ChannelInboundInvoker, ChannelOutboundInvoker, Iterable<Entry<String, ChannelHandler>> {
    // Handler 추가
    ChannelPipeline addFirst(String name, ChannelHandler handler);
    ChannelPipeline addLast(String name, ChannelHandler handler);
    ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
    ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);

    // Handler 제거
    ChannelPipeline remove(ChannelHandler handler);
    ChannelHandler remove(String name);
    <T extends ChannelHandler> T remove(Class<T> handlerType);

    // Handler 교체
    ChannelPipeline replace(ChannelHandler oldHandler, String newName, ChannelHandler newHandler);

    // Handler 조회
    ChannelHandler get(String name);
    <T extends ChannelHandler> T get(Class<T> handlerType);
    ChannelHandlerContext context(ChannelHandler handler);

    Channel channel();
}
```

#### DefaultChannelPipeline (구체 클래스)
**역할**: 파이프라인 구현

**구조**:
```java
public class DefaultChannelPipeline implements ChannelPipeline {
    final AbstractChannelHandlerContext head;  // HeadContext (Outbound)
    final AbstractChannelHandlerContext tail;  // TailContext (Inbound)

    private final Channel channel;

    protected DefaultChannelPipeline(Channel channel) {
        this.channel = channel;

        // 양방향 링크드 리스트
        tail = new TailContext(this);
        head = new HeadContext(this);
        head.next = tail;
        tail.prev = head;
    }
}
```

#### ChannelHandler (인터페이스)
**역할**: 이벤트 처리의 기본 단위

```java
public interface ChannelHandler {
    void handlerAdded(ChannelHandlerContext ctx) throws Exception;
    void handlerRemoved(ChannelHandlerContext ctx) throws Exception;

    @Deprecated
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;

    @Sharable  // 어노테이션: 여러 파이프라인에서 공유 가능
    // 공유 가능한 Handler는 상태를 가지면 안 됨
}
```

#### ChannelInboundHandler (인터페이스)
**역할**: Inbound 이벤트 처리 (네트워크 → 애플리케이션)

```java
public interface ChannelInboundHandler extends ChannelHandler {
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
    void channelActive(ChannelHandlerContext ctx) throws Exception;
    void channelInactive(ChannelHandlerContext ctx) throws Exception;

    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;  // 데이터 수신
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;

    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;

    @Override
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
```

#### ChannelOutboundHandler (인터페이스)
**역할**: Outbound 이벤트 처리 (애플리케이션 → 네트워크)

```java
public interface ChannelOutboundHandler extends ChannelHandler {
    void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise);
    void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress,
                 SocketAddress localAddress, ChannelPromise promise);
    void disconnect(ChannelHandlerContext ctx, ChannelPromise promise);
    void close(ChannelHandlerContext ctx, ChannelPromise promise);
    void deregister(ChannelHandlerContext ctx, ChannelPromise promise);

    void read(ChannelHandlerContext ctx);
    void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise);  // 데이터 송신
    void flush(ChannelHandlerContext ctx);
}
```

#### ChannelHandlerContext (인터페이스)
**역할**: Handler와 Pipeline 간 상호작용

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {
    Channel channel();
    EventExecutor executor();
    String name();
    ChannelHandler handler();
    boolean isRemoved();

    // Inbound 전파
    ChannelHandlerContext fireChannelRegistered();
    ChannelHandlerContext fireChannelActive();
    ChannelHandlerContext fireChannelRead(Object msg);
    ChannelHandlerContext fireChannelReadComplete();
    ChannelHandlerContext fireExceptionCaught(Throwable cause);

    // Outbound 전파
    ChannelFuture bind(SocketAddress localAddress);
    ChannelFuture connect(SocketAddress remoteAddress);
    ChannelFuture write(Object msg);
    ChannelHandlerContext flush();
    ChannelFuture writeAndFlush(Object msg);

    ChannelPipeline pipeline();
    ByteBufAllocator alloc();
}
```

#### 파이프라인 처리 흐름 다이어그램

```
┌──────────────────────────────────────────────────────────────┐
│                   ChannelPipeline 구조                        │
└──────────────────────────────────────────────────────────────┘

Inbound 이벤트 (데이터 읽기):
Head → Handler1 (I) → Handler2 (I) → Handler3 (I) → Tail
 ↑                                                      ↓
 │                                            (TailContext:
 │                                             기본 처리, 경고)
 │
(HeadContext: I/O 작업)

Outbound 이벤트 (데이터 쓰기):
Tail ← Handler3 (O) ← Handler2 (O) ← Handler1 (O) ← Head
 ↓                                                      ↑
 │                                              (애플리케이션
(무시)                                           write() 호출)

양방향 Handler:
Handler implements ChannelDuplexHandler
  - Inbound: Head → ... → Handler → ... → Tail
  - Outbound: Tail → ... → Handler → ... → Head

전파 메서드:
- Inbound: ctx.fireChannelRead(msg)
- Outbound: ctx.write(msg)
```

#### ChannelInitializer (추상 클래스)
**역할**: 파이프라인 초기 설정

```java
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        // 한 번만 실행되고 자동 제거
        if (initChannel(ctx)) {
            ctx.pipeline().remove(this);
            ctx.fireChannelRegistered();
        } else {
            ctx.fireChannelRegistered();
        }
    }
}
```

**사용 예제**:
```java
new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();

        // Inbound 순서
        p.addLast("decoder", new StringDecoder());
        p.addLast("handler", new BusinessHandler());

        // Outbound 순서 (역방향)
        p.addLast("encoder", new StringEncoder());
    }
}
```

### 4.5 ChannelFuture & Promise - 비동기 작업 관리

#### 파일 위치
```
common/src/main/java/io/netty/util/concurrent/
├── Future.java                  (인터페이스)
├── Promise.java                 (인터페이스)
└── DefaultPromise.java          (구현)

transport/src/main/java/io/netty/channel/
├── ChannelFuture.java           (인터페이스)
├── ChannelPromise.java          (인터페이스)
└── DefaultChannelPromise.java   (구현)
```

#### ChannelFuture (인터페이스)
**역할**: 비동기 작업의 결과 표현

```java
public interface ChannelFuture extends Future<Void> {
    Channel channel();  // 관련 Channel

    // 리스너 등록
    ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> listener);
    ChannelFuture removeListener(GenericFutureListener<? extends Future<? super Void>> listener);

    // 동기 대기 (권장하지 않음)
    ChannelFuture sync() throws InterruptedException;
    ChannelFuture syncUninterruptibly();
    ChannelFuture await() throws InterruptedException;

    // 상태 확인
    boolean isSuccess();
    boolean isCancellable();
    Throwable cause();
}
```

#### ChannelPromise (인터페이스)
**역할**: 쓰기 가능한 ChannelFuture

```java
public interface ChannelPromise extends ChannelFuture, Promise<Void> {
    // 결과 설정
    ChannelPromise setSuccess(Void result);
    ChannelPromise setSuccess();
    boolean trySuccess();
    ChannelPromise setFailure(Throwable cause);
    boolean tryFailure(Throwable cause);

    // Void가 아닌 setSuccess를 사용할 수 없음
    boolean setUncancellable();
}
```

**사용 예제**:
```java
// 콜백 방식 (권장)
ChannelFuture future = ctx.writeAndFlush(msg);
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture f) {
        if (f.isSuccess()) {
            System.out.println("쓰기 성공");
        } else {
            System.err.println("쓰기 실패: " + f.cause());
            f.channel().close();
        }
    }
});

// 동기 방식 (비권장, EventLoop 블로킹 주의)
ChannelFuture future = ctx.writeAndFlush(msg);
future.sync();  // 완료 대기
```

### 4.6 핵심 아키텍처 종합 다이어그램

#### 새로운 아키텍처 (권장)

> ✅ **Netty 4.2 권장 방식**: 최신 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                  Netty 핵심 아키텍처 (New)                       │
└─────────────────────────────────────────────────────────────────┘

애플리케이션 레이어:
┌──────────────────────────────────────────────────────────────┐
│  ServerBootstrap                                             │
│    .group(bossGroup, workerGroup)                            │
│    .channel(NioServerSocketChannel.class)                    │
│    .childHandler(new ChannelInitializer<SocketChannel>() {   │
│        public void initChannel(SocketChannel ch) {           │
│            ch.pipeline().addLast(new MyHandler());           │
│        }                                                      │
│    })                                                         │
│    .bind(8080);                                              │
└──────────────────────────────────────────────────────────────┘
                           │
                           ↓
EventLoopGroup 레이어:
┌──────────────────────────────────────────────────────────────┐
│  bossGroup (MultiThreadIoEventLoopGroup)                     │
│    └─ IoEventLoop-1 → NioIoHandler → Selector (OP_ACCEPT)   │
│                                                               │
│  workerGroup (MultiThreadIoEventLoopGroup)                   │
│    ├─ IoEventLoop-1 → NioIoHandler → Selector                │
│    ├─ IoEventLoop-2 → NioIoHandler → Selector                │
│    └─ IoEventLoop-N → NioIoHandler → Selector                │
│                                                               │
│  IoHandler 추상화 (선택 가능):                                │
│    - NioIoHandler: Java NIO Selector                         │
│    - EpollIoHandler: Linux epoll                             │
│    - KQueueIoHandler: macOS/BSD kqueue                       │
│    - IoUringIoHandler: Linux io_uring                        │
└──────────────────────────────────────────────────────────────┘
                           │
                           ↓
Channel 레이어:
┌──────────────────────────────────────────────────────────────┐
│  NioServerSocketChannel (parent)                             │
│    └─ accept() → NioSocketChannel[] (children)               │
│                                                               │
│  NioSocketChannel (각 클라이언트 연결)                        │
│    ├─ ChannelConfig (옵션)                                   │
│    ├─ ChannelPipeline (핸들러 체인)                          │
│    └─ IoEventLoop (할당된 스레드)                            │
└──────────────────────────────────────────────────────────────┘
                           │
                           ↓
ChannelPipeline 레이어:
┌──────────────────────────────────────────────────────────────┐
│  Head → [Codec] → [Handler] → ... → Tail                     │
│                                                               │
│  Inbound: Head → ... → Tail                                  │
│  Outbound: Tail → ... → Head                                 │
└──────────────────────────────────────────────────────────────┘
                           │
                           ↓
I/O 레이어:
┌──────────────────────────────────────────────────────────────┐
│  IoHandler에 의해 추상화된 I/O:                              │
│    - Java NIO (Selector, SelectionKey, SocketChannel)        │
│    - Native epoll (Linux)                                    │
│    - Native kqueue (macOS/BSD)                               │
│    - Native io_uring (Linux 5.1+)                            │
└──────────────────────────────────────────────────────────────┘

핵심 개선사항:
✅ IoHandler를 통한 Transport 추상화
✅ 코드 중복 제거 (공통 로직 MultiThreadIoEventLoopGroup에서 관리)
✅ 확장성 향상 (커스텀 IoHandler 구현 가능)
```

#### 기존 아키텍처 (Deprecated)

> ⚠️ **Deprecated**: 레거시 참고용

```
┌─────────────────────────────────────────────────────────────────┐
│                  Netty 핵심 아키텍처 (Legacy)                    │
└─────────────────────────────────────────────────────────────────┘

애플리케이션 레이어:
┌──────────────────────────────────────────────────────────────┐
│  ServerBootstrap                                             │
│    .group(bossGroup, workerGroup)                            │
│    .channel(NioServerSocketChannel.class)                    │
│    .childHandler(...)                                        │
│    .bind(8080);                                              │
└──────────────────────────────────────────────────────────────┘
                           │
                           ↓
EventLoopGroup 레이어:
┌──────────────────────────────────────────────────────────────┐
│  bossGroup (NioEventLoopGroup) - Deprecated                  │
│    └─ NioEventLoop-1 → Selector (OP_ACCEPT)                  │
│                                                               │
│  workerGroup (NioEventLoopGroup) - Deprecated                │
│    ├─ NioEventLoop-1 → Selector (OP_READ, OP_WRITE)          │
│    ├─ NioEventLoop-2 → Selector                              │
│    └─ NioEventLoop-N → Selector                              │
└──────────────────────────────────────────────────────────────┘

문제점:
❌ Transport별로 별도 클래스 필요 (NioEventLoopGroup, EpollEventLoopGroup 등)
❌ 코드 중복 (각 Transport마다 유사한 로직 반복)
❌ 확장 어려움 (새 Transport 추가 시 전체 구조 복제)
```

### 4.7 Phase 1 체크리스트

- [ ] Bootstrap과 ServerBootstrap의 차이 이해
- [ ] Channel 상태 전이(Registered → Active → Inactive) 이해
- [ ] EventLoop의 단일 스레드 특성 이해
- [ ] ChannelPipeline의 Inbound/Outbound 방향 이해
- [ ] ChannelHandlerContext를 통한 이벤트 전파 이해
- [ ] ChannelFuture를 사용한 비동기 처리 이해

---

## 5. Phase 2: 메모리 관리

> **목표**: Netty의 고성능 메모리 관리 시스템 이해

### 5.1 ByteBuf - Netty의 바이트 버퍼

#### 파일 위치
```
buffer/src/main/java/io/netty/buffer/
├── ByteBuf.java                 (인터페이스)
├── AbstractByteBuf.java         (추상 구현)
├── AbstractReferenceCountedByteBuf.java  (참조 카운팅)
├── UnpooledHeapByteBuf.java     (힙 버퍼)
├── UnpooledDirectByteBuf.java   (다이렉트 버퍼)
├── PooledHeapByteBuf.java       (풀링된 힙 버퍼)
└── PooledDirectByteBuf.java     (풀링된 다이렉트 버퍼)
```

#### ByteBuf (인터페이스)
**역할**: Java ByteBuffer의 향상된 대안

**핵심 특징**:
1. **두 개의 포인터**: readerIndex, writerIndex
2. **용량 관리**: capacity, maxCapacity
3. **참조 카운팅**: retain(), release()
4. **파생 버퍼**: slice(), duplicate(), copy()

**주요 메서드**:
```java
public interface ByteBuf extends ReferenceCounted, Comparable<ByteBuf> {
    // 용량
    int capacity();
    ByteBuf capacity(int newCapacity);
    int maxCapacity();

    // 인덱스
    int readerIndex();
    ByteBuf readerIndex(int readerIndex);
    int writerIndex();
    ByteBuf writerIndex(int writerIndex);
    ByteBuf setIndex(int readerIndex, int writerIndex);

    int readableBytes();  // writerIndex - readerIndex
    int writableBytes();  // capacity - writerIndex
    int maxWritableBytes();
    boolean isReadable();
    boolean isWritable();

    ByteBuf clear();  // readerIndex = writerIndex = 0
    ByteBuf markReaderIndex();
    ByteBuf resetReaderIndex();

    // 읽기 (readerIndex 증가)
    byte readByte();
    short readShort();
    int readInt();
    long readLong();
    ByteBuf readBytes(byte[] dst);
    ByteBuf readBytes(ByteBuf dst);

    // 쓰기 (writerIndex 증가)
    ByteBuf writeByte(int value);
    ByteBuf writeShort(int value);
    ByteBuf writeInt(int value);
    ByteBuf writeLong(long value);
    ByteBuf writeBytes(byte[] src);
    ByteBuf writeBytes(ByteBuf src);

    // Get/Set (인덱스 변경 없음)
    byte getByte(int index);
    ByteBuf setByte(int index, int value);

    // 파생 버퍼
    ByteBuf slice();  // readerIndex ~ writerIndex 공유
    ByteBuf slice(int index, int length);
    ByteBuf duplicate();  // 전체 공유
    ByteBuf copy();  // 복사본

    // 참조 카운팅 (ReferenceCounted)
    int refCnt();
    ByteBuf retain();
    ByteBuf retain(int increment);
    boolean release();
    boolean release(int decrement);

    // 타입
    boolean hasArray();  // Heap 여부
    byte[] array();
    int arrayOffset();
    boolean hasMemoryAddress();  // Direct 여부
    long memoryAddress();

    // 기타
    ByteBufAllocator alloc();
    ByteOrder order();
    boolean isDirect();
}
```

#### ByteBuf vs Java ByteBuffer 비교

```
┌──────────────────────────────────────────────────────────────┐
│                    ByteBuf vs ByteBuffer                      │
└──────────────────────────────────────────────────────────────┘

Java ByteBuffer:
┌────────────────────────────────────┐
│  0  ≤  position  ≤  limit  ≤  cap  │
└────────────────────────────────────┘
- flip() 필요 (읽기/쓰기 모드 전환)
- 용량 고정
- 참조 카운팅 없음

Netty ByteBuf:
┌───────────────────────────────────────────────────┐
│  0  ≤  readerIndex  ≤  writerIndex  ≤  capacity   │
└───────────────────────────────────────────────────┘
- 동시 읽기/쓰기
- 동적 확장 (maxCapacity까지)
- 참조 카운팅 (메모리 누수 방지)
```

#### ByteBuf 메모리 레이아웃

```
┌──────────────────────────────────────────────────────────────┐
│                     ByteBuf 메모리 구조                       │
└──────────────────────────────────────────────────────────────┘

  0         readerIndex       writerIndex          capacity
  ├───────────┼─────────────────┼───────────────────┤
  │ discarded │   readable      │     writable      │
  │   bytes   │     bytes       │      bytes        │
  └───────────┴─────────────────┴───────────────────┘

읽기 작업:
- readByte() → readerIndex++
- readableBytes() = writerIndex - readerIndex

쓰기 작업:
- writeByte() → writerIndex++
- writableBytes() = capacity - writerIndex

확장:
- writerIndex == capacity → ensureWritable() → realloc
- maxCapacity 제한

정리:
- discardReadBytes() → 읽은 부분 버림, readerIndex = 0
```

### 5.2 ByteBufAllocator - 메모리 할당자

#### 파일 위치
```
buffer/src/main/java/io/netty/buffer/
├── ByteBufAllocator.java             (인터페이스)
├── AbstractByteBufAllocator.java     (추상 구현)
├── PooledByteBufAllocator.java       (풀링 할당자, 기본)
└── UnpooledByteBufAllocator.java     (비풀링 할당자)
```

#### ByteBufAllocator (인터페이스)
**역할**: ByteBuf 생성 팩토리

```java
public interface ByteBufAllocator {
    // 기본 할당자
    ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;  // PooledByteBufAllocator

    // 일반 버퍼 (Heap 또는 Direct, 자동 선택)
    ByteBuf buffer();
    ByteBuf buffer(int initialCapacity);
    ByteBuf buffer(int initialCapacity, int maxCapacity);

    // I/O 버퍼 (일반적으로 Direct)
    ByteBuf ioBuffer();
    ByteBuf ioBuffer(int initialCapacity);

    // Heap 버퍼
    ByteBuf heapBuffer();
    ByteBuf heapBuffer(int initialCapacity);
    ByteBuf heapBuffer(int initialCapacity, int maxCapacity);

    // Direct 버퍼
    ByteBuf directBuffer();
    ByteBuf directBuffer(int initialCapacity);
    ByteBuf directBuffer(int initialCapacity, int maxCapacity);

    // Composite 버퍼
    CompositeByteBuf compositeBuffer();
    CompositeByteBuf compositeBuffer(int maxNumComponents);
    CompositeByteBuf compositeHeapBuffer();
    CompositeByteBuf compositeDirectBuffer();

    // 용량 계산
    int calculateNewCapacity(int minNewCapacity, int maxCapacity);

    // 타입 확인
    boolean isDirectBufferPooled();
}
```

#### PooledByteBufAllocator (구체 클래스)
**역할**: jemalloc 기반 메모리 풀 관리

**주요 필드**:
```java
public class PooledByteBufAllocator extends AbstractByteBufAllocator implements ByteBufAllocatorMetricProvider {
    private final PoolArena<byte[]>[] heapArenas;      // Heap 메모리 풀
    private final PoolArena<ByteBuffer>[] directArenas; // Direct 메모리 풀
    private final PoolThreadLocalCache threadCache;     // 스레드별 캐시

    // 설정
    public static final int DEFAULT_NUM_HEAP_ARENA =
        Math.max(0, SystemPropertyUtil.getInt("io.netty.allocator.numHeapArenas",
                                               (int) Math.min(defaultMinNumArena,
                                                              PlatformDependent.estimateMaxDirectMemory() / defaultChunkSize / 2 / 3)));
    public static final int DEFAULT_PAGE_SIZE = 8192;  // 8KB
    public static final int DEFAULT_MAX_ORDER = 9;     // 2^9 = 512 pages = 4MB chunk
    public static final int DEFAULT_SMALL_CACHE_SIZE = 256;
    public static final int DEFAULT_NORMAL_CACHE_SIZE = 64;
}
```

### 5.3 참조 카운팅 (Reference Counting)

#### 파일 위치
```
common/src/main/java/io/netty/util/
├── ReferenceCounted.java        (인터페이스)
└── internal/
    └── RefCnt.java              (구현)

buffer/src/main/java/io/netty/buffer/
└── AbstractReferenceCountedByteBuf.java
```

#### ReferenceCounted (인터페이스)
**역할**: 명시적 메모리 관리

```java
public interface ReferenceCounted {
    int refCnt();  // 현재 참조 카운트

    ReferenceCounted retain();  // +1
    ReferenceCounted retain(int increment);

    ReferenceCounted touch();
    ReferenceCounted touch(Object hint);

    boolean release();  // -1, 0이면 true 반환 및 메모리 해제
    boolean release(int decrement);
}
```

#### 참조 카운팅 규칙

```
┌──────────────────────────────────────────────────────────────┐
│                   참조 카운팅 생명주기                        │
└──────────────────────────────────────────────────────────────┘

1. 생성 시:
   ByteBuf buf = allocator.buffer();  // refCnt = 1

2. 전달 시 (소유권 이전):
   ctx.write(buf);  // Handler가 자동으로 release()
   // 주의: write 후 buf 사용 금지!

3. 보유 시 (참조 추가):
   ByteBuf copy = buf.retain();  // refCnt = 2
   // 나중에 copy.release() 필수

4. 해제 시:
   buf.release();  // refCnt = 0 → 메모리 반환

메모리 누수 경고:
io.netty.util.ResourceLeakDetector - LEAK: ByteBuf.release() was not called
  at io.netty.buffer.AdvancedLeakAwareByteBuf.leak(...)
```

#### 일반적인 패턴

```java
// 패턴 1: 자동 해제 (Handler에서)
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf buf = (ByteBuf) msg;
    try {
        // 처리
        process(buf);
    } finally {
        buf.release();  // 필수!
    }
}

// 패턴 2: 전달 (다음 Handler로)
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf buf = (ByteBuf) msg;
    // 처리
    transform(buf);
    ctx.fireChannelRead(buf);  // 소유권 이전 (release 하지 않음!)
}

// 패턴 3: 보유 (나중에 사용)
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf buf = (ByteBuf) msg;
    this.savedBuf = buf.retain();  // 참조 추가
    ctx.fireChannelRead(buf);
}

@Override
public void channelInactive(ChannelHandlerContext ctx) {
    if (savedBuf != null) {
        savedBuf.release();  // 나중에 해제
    }
}

// 패턴 4: SimpleChannelInboundHandler (자동 해제)
public class MyHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
        // 처리
        // release() 자동 호출됨!
    }
}
```

### 5.4 메모리 풀 아키텍처

#### PoolArena - 메모리 영역

**파일**: `buffer/src/main/java/io/netty/buffer/PoolArena.java`

```java
abstract class PoolArena<T> implements PoolArenaMetric {
    final PooledByteBufAllocator parent;

    // 크기 클래스
    final int numSmallSubpagePools;
    final int directMemoryCacheAlignment;

    // SubPage 풀 (작은 할당용)
    private final PoolSubpage<T>[] smallSubpagePools;

    // Chunk 리스트 (사용률별)
    private final PoolChunkList<T> q050;  // 25-75% 사용률
    private final PoolChunkList<T> q025;  // 1-50%
    private final PoolChunkList<T> q000;  // 1-25%
    private final PoolChunkList<T> qInit; // 0-25%
    private final PoolChunkList<T> q075;  // 50-100%
    private final PoolChunkList<T> q100;  // 100%

    PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
        PooledByteBuf<T> buf = newByteBuf(maxCapacity);
        allocate(cache, buf, reqCapacity);
        return buf;
    }
}
```

#### 메모리 할당 계층 구조

```
┌──────────────────────────────────────────────────────────────┐
│                  메모리 할당 계층 구조                        │
└──────────────────────────────────────────────────────────────┘

PooledByteBufAllocator
    │
    ├─ PoolArena[] (Heap)
    │   ├─ PoolChunkList (qInit, q000, q025, q050, q075, q100)
    │   │   └─ PoolChunk[] (4MB 청크)
    │   │       └─ Page[] (8KB 페이지)
    │   └─ PoolSubpage[] (작은 할당용, < 8KB)
    │
    ├─ PoolArena[] (Direct)
    │   └─ (동일 구조)
    │
    └─ PoolThreadCache (스레드 로컬)
        ├─ MemoryRegionCache[] (Small Heap)
        ├─ MemoryRegionCache[] (Small Direct)
        ├─ MemoryRegionCache[] (Normal Heap)
        └─ MemoryRegionCache[] (Normal Direct)

할당 크기 분류:
- Tiny: < 512B
- Small: 512B ~ 8KB (pageSize)
- Normal: 8KB ~ 4MB (chunkSize)
- Huge: > 4MB (직접 할당, 풀링 없음)
```

#### PoolChunk - 청크 관리

**파일**: `buffer/src/main/java/io/netty/buffer/PoolChunk.java`

```
PoolChunk 구조:
┌─────────────────────────────────────────────────────────────┐
│                    Chunk (4MB, 2^9 pages)                    │
├─────────────────────────────────────────────────────────────┤
│  Page 0 (8KB)  │  Page 1  │  ...  │  Page 511  │            │
└─────────────────────────────────────────────────────────────┘

Handle 인코딩 (64bit):
- runOffset (15bit): 페이지 오프셋
- size/pages (15bit): 크기
- isUsed (1bit): 사용 중
- isSubpage (1bit): SubPage 여부
- bitmapIdx (32bit): SubPage 비트맵 인덱스

jemalloc 알고리즘:
- Buddy allocation (이진 트리)
- 외부 단편화 최소화
```

#### PoolThreadCache - 스레드 캐시

**파일**: `buffer/src/main/java/io/netty/buffer/PoolThreadCache.java`

```java
final class PoolThreadCache {
    final PoolArena<byte[]> heapArena;
    final PoolArena<ByteBuffer> directArena;

    // 크기별 캐시
    private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;
    private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
    private final MemoryRegionCache<byte[]>[] normalHeapCaches;
    private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;

    // 정리 주기
    private int freeSweepAllocationThreshold;
    private final AtomicBoolean freed = new AtomicBoolean();
}
```

**캐시 전략**:
- 할당 시: 캐시 조회 → Arena 할당
- 해제 시: 캐시 추가 → 가득 차면 Arena 반환
- Lock-free (스레드 로컬)

### 5.5 Heap vs Direct 버퍼

```
┌──────────────────────────────────────────────────────────────┐
│                   Heap vs Direct 버퍼                         │
└──────────────────────────────────────────────────────────────┘

Heap 버퍼 (UnpooledHeapByteBuf, PooledHeapByteBuf):
┌────────────────────────────────────────────────┐
│  byte[] array (Java 힙 메모리)                 │
└────────────────────────────────────────────────┘
- GC 관리
- 빠른 할당/해제
- I/O 시 임시 Direct 버퍼로 복사
- hasArray() = true
- array() 접근 가능

Direct 버퍼 (UnpooledDirectByteBuf, PooledDirectByteBuf):
┌────────────────────────────────────────────────┐
│  ByteBuffer (네이티브 메모리)                  │
└────────────────────────────────────────────────┘
- GC 외부 (Cleaner로 해제)
- 할당 비용 높음 → 풀링 필수
- I/O 효율적 (zero-copy)
- hasMemoryAddress() = true
- memoryAddress() 접근 가능

성능 권장사항:
- I/O 집약적: Direct 버퍼 (기본값)
- 메모리 집약적: Heap 버퍼
- 대부분: PooledByteBufAllocator 사용 (기본값)
```

### 5.6 CompositeByteBuf - 복합 버퍼

**파일**: `buffer/src/main/java/io/netty/buffer/CompositeByteBuf.java`

**역할**: 여러 ByteBuf를 논리적으로 결합 (zero-copy)

```java
public class CompositeByteBuf extends AbstractReferenceCountedByteBuf implements Iterable<ByteBuf> {
    private final ByteBufAllocator alloc;
    private final boolean direct;
    private final int maxNumComponents;
    private final ComponentList components;

    // 컴포넌트 추가
    public CompositeByteBuf addComponent(ByteBuf buffer);
    public CompositeByteBuf addComponents(ByteBuf... buffers);

    // 인덱스 매핑
    private Component findComponent(int offset);

    // 최적화
    public CompositeByteBuf consolidate();  // 단일 버퍼로 병합
}
```

**사용 예제**:
```java
// HTTP 헤더 + 바디 결합 (복사 없음)
ByteBuf header = allocator.buffer(128);
ByteBuf body = allocator.buffer(1024);

CompositeByteBuf httpMessage = allocator.compositeBuffer();
httpMessage.addComponent(true, header);  // true: writerIndex 증가
httpMessage.addComponent(true, body);

ctx.write(httpMessage);  // 단일 버퍼처럼 전송
```

### 5.7 Phase 2 체크리스트

- [ ] ByteBuf의 readerIndex/writerIndex 동작 이해
- [ ] 참조 카운팅 규칙 (retain/release) 숙지
- [ ] PooledByteBufAllocator vs UnpooledByteBufAllocator 차이
- [ ] Heap vs Direct 버퍼 선택 기준
- [ ] 메모리 누수 디버깅 방법
- [ ] CompositeByteBuf 사용 시기

---

## 6. Phase 3: 프로토콜 코덱

> **목표**: 다양한 프로토콜 지원 구조 이해

### 6.1 코덱 기본 구조

#### 파일 위치
```
codec-base/src/main/java/io/netty/handler/codec/
├── ByteToMessageDecoder.java         (기본 디코더)
├── MessageToByteEncoder.java         (기본 인코더)
├── MessageToMessageDecoder.java      (메시지 변환 디코더)
├── MessageToMessageEncoder.java      (메시지 변환 인코더)
└── ReplayingDecoder.java             (상태 기반 디코더)
```

#### ByteToMessageDecoder (추상 클래스)
**역할**: 바이트 스트림 → 메시지 객체

```java
public abstract class ByteToMessageDecoder extends ChannelInboundHandlerAdapter {
    ByteBuf cumulation;  // 누적 버퍼
    private Cumulator cumulator = MERGE_CUMULATOR;  // 누적 전략

    // 구현 필수
    protected abstract void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
        throws Exception;

    // 선택 구현
    protected void decodeLast(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
        throws Exception {
        decode(ctx, in, out);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf data = (ByteBuf) msg;
        try {
            // 누적
            cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);

            // 디코딩 (여러 메시지 가능)
            callDecode(ctx, cumulation, out);
        } finally {
            if (cumulation.isReadable()) {
                // 읽지 않은 데이터 유지
                cumulation.discardSomeReadBytes();
            } else {
                cumulation.release();
                cumulation = null;
            }
        }
    }
}
```

**사용 예제**:
```java
// 고정 길이 메시지 디코더
public class FixedLengthFrameDecoder extends ByteToMessageDecoder {
    private final int frameLength;

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        if (in.readableBytes() < frameLength) {
            return;  // 데이터 부족, 더 기다림
        }
        out.add(in.readRetainedSlice(frameLength));  // 프레임 추출
    }
}
```

#### MessageToByteEncoder (추상 클래스)
**역할**: 메시지 객체 → 바이트 스트림

```java
public abstract class MessageToByteEncoder<I> extends ChannelOutboundHandlerAdapter {
    private final TypeParameterMatcher matcher;
    private final boolean preferDirect;

    // 구현 필수
    protected abstract void encode(ChannelHandlerContext ctx, I msg, ByteBuf out)
        throws Exception;

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)
        throws Exception {
        ByteBuf buf = null;
        try {
            if (acceptOutboundMessage(msg)) {
                I cast = (I) msg;
                buf = allocateBuffer(ctx, cast, preferDirect);  // 버퍼 할당
                encode(ctx, cast, buf);  // 인코딩

                ReferenceCountUtil.release(msg);  // 원본 해제
                ctx.write(buf, promise);  // 전송
                buf = null;
            } else {
                ctx.write(msg, promise);  // 타입 불일치, 패스
            }
        } finally {
            if (buf != null) {
                buf.release();
            }
        }
    }
}
```

**사용 예제**:
```java
// 정수 인코더 (4바이트)
public class IntegerEncoder extends MessageToByteEncoder<Integer> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) {
        out.writeInt(msg);
    }
}
```

### 6.2 HTTP 코덱

#### 파일 위치
```
codec-http/src/main/java/io/netty/handler/codec/http/
├── HttpServerCodec.java          (서버 코덱)
├── HttpClientCodec.java          (클라이언트 코덱)
├── HttpRequestDecoder.java       (요청 디코더)
├── HttpResponseEncoder.java      (응답 인코더)
├── HttpObjectAggregator.java     (메시지 조립)
└── websocketx/
    ├── WebSocketServerHandshaker.java
    └── WebSocket13FrameDecoder.java
```

#### HttpServerCodec (구체 클래스)
**역할**: HTTP 요청 디코더 + 응답 인코더 결합

```java
public final class HttpServerCodec extends CombinedChannelDuplexHandler
        <HttpRequestDecoder, HttpResponseEncoder>
        implements HttpServerUpgradeHandler.SourceCodec {

    private final Queue<HttpMethod> queue = new ArrayDeque<HttpMethod>();

    public HttpServerCodec(HttpDecoderConfig config) {
        init(new HttpRequestDecoder(config), new HttpResponseEncoder());
    }
}
```

#### HTTP 메시지 계층

```
HttpObject (루트)
    ├─ HttpMessage (헤더)
    │   ├─ HttpRequest
    │   │   └─ FullHttpRequest (전체 요청)
    │   └─ HttpResponse
    │       └─ FullHttpResponse (전체 응답)
    └─ HttpContent (바디)
        ├─ DefaultHttpContent
        └─ LastHttpContent (종료 마커)
            └─ EmptyLastHttpContent
```

#### HttpObjectAggregator (구체 클래스)
**역할**: HTTP 메시지 + Content 조각들 → FullHttpMessage

```java
public class HttpObjectAggregator extends MessageAggregator
        <HttpObject, HttpMessage, HttpContent, FullHttpMessage> {

    private final int maxContentLength;

    @Override
    protected FullHttpMessage beginAggregation(HttpMessage start, ByteBuf content) {
        // HttpRequest + Content → FullHttpRequest
        if (start instanceof HttpRequest) {
            return new DefaultFullHttpRequest(...);
        } else if (start instanceof HttpResponse) {
            return new DefaultFullHttpResponse(...);
        }
    }
}
```

**파이프라인 구성**:
```java
ch.pipeline().addLast("codec", new HttpServerCodec());
ch.pipeline().addLast("aggregator", new HttpObjectAggregator(1048576));  // 1MB
ch.pipeline().addLast("handler", new HttpServerHandler());
```

### 6.3 HTTP/2 코덱

#### 파일 위치
```
codec-http2/src/main/java/io/netty/handler/codec/http2/
├── Http2FrameCodec.java              (프레임 코덱)
├── Http2MultiplexHandler.java        (스트림 멀티플렉싱)
├── Http2ConnectionHandler.java       (연결 핸들러)
├── Http2FrameReader.java             (프레임 읽기)
├── Http2FrameWriter.java             (프레임 쓰기)
└── HpackEncoder.java                 (헤더 압축)
```

#### Http2FrameCodec (구체 클래스)
**역할**: HTTP/2 프레임 처리

**프레임 타입**:
- `Http2DataFrame`: 데이터
- `Http2HeadersFrame`: 헤더
- `Http2SettingsFrame`: 설정
- `Http2WindowUpdateFrame`: 흐름 제어
- `Http2PingFrame`: Keep-alive
- `Http2GoAwayFrame`: 연결 종료

**사용 예제**:
```java
Http2FrameCodecBuilder.forServer()
    .initialSettings(Http2Settings.defaultSettings())
    .build();
```

#### Http2MultiplexHandler (구체 클래스)
**역할**: HTTP/2 스트림 멀티플렉싱

```java
public final class Http2MultiplexHandler extends Http2ChannelDuplexHandler {
    private final ChannelHandler inboundStreamHandler;
    private final ChannelHandler upgradeStreamHandler;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof Http2StreamFrame) {
            Http2StreamFrame frame = (Http2StreamFrame) msg;
            // 스트림별 Channel로 라우팅
            Http2FrameStream stream = frame.stream();
            // ...
        }
    }
}
```

### 6.4 WebSocket 코덱

#### WebSocket 핸드셰이크

```
클라이언트 → 서버:
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

서버 → 클라이언트:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

#### WebSocketServerHandshaker (추상 클래스)
**역할**: 핸드셰이크 처리

```java
public abstract class WebSocketServerHandshaker {
    protected abstract FullHttpResponse newHandshakeResponse(FullHttpRequest req,
                                                              HttpHeaders headers);

    public ChannelFuture handshake(Channel channel, FullHttpRequest req) {
        return handshake(channel, req, null, channel.newPromise());
    }
}
```

**사용 예제**:
```java
WebSocketServerHandshakerFactory factory =
    new WebSocketServerHandshakerFactory("ws://localhost:8080/ws", null, true);

WebSocketServerHandshaker handshaker = factory.newHandshaker(request);
if (handshaker == null) {
    WebSocketServerHandshakerFactory.sendUnsupportedVersionResponse(ctx.channel());
} else {
    handshaker.handshake(ctx.channel(), request);
}
```

#### WebSocket 프레임 타입

```
WebSocketFrame
    ├─ TextWebSocketFrame: UTF-8 텍스트
    ├─ BinaryWebSocketFrame: 바이너리
    ├─ ContinuationWebSocketFrame: 연속 프레임
    ├─ CloseWebSocketFrame: 연결 종료
    ├─ PingWebSocketFrame: Ping
    └─ PongWebSocketFrame: Pong
```

### 6.5 기타 프로토콜

#### Redis (RESP)
**파일**: `codec-redis/src/main/java/io/netty/handler/codec/redis/RedisDecoder.java`

```java
public final class RedisDecoder extends ByteToMessageDecoder {
    enum State {
        DECODE_TYPE,          // +, -, :, $, *
        DECODE_INLINE,        // Simple String, Error, Integer
        DECODE_LENGTH,        // Bulk String, Array
        DECODE_BULK_STRING_EOL,
        DECODE_BULK_STRING_CONTENT
    }
}
```

#### MQTT (IoT)
**파일**: `codec-mqtt/src/main/java/io/netty/handler/codec/mqtt/MqttDecoder.java`

```java
public final class MqttDecoder extends ReplayingDecoder<DecoderState> {
    enum DecoderState {
        READ_FIXED_HEADER,
        READ_VARIABLE_LENGTH,
        READ_PAYLOAD
    }
}
```

### 6.6 코덱 구성 전략

#### 일반적인 파이프라인

```
┌──────────────────────────────────────────────────────────────┐
│                     일반적인 파이프라인                       │
└──────────────────────────────────────────────────────────────┘

서버:
┌────────────────────────────────────────┐
│  1. LengthFieldBasedFrameDecoder       │  (프레임 분리)
│  2. ProtobufDecoder                    │  (역직렬화)
│  3. BusinessHandler                    │  (비즈니스 로직)
│  4. ProtobufEncoder                    │  (직렬화)
│  5. LengthFieldPrepender               │  (길이 필드 추가)
└────────────────────────────────────────┘

HTTP 서버:
┌────────────────────────────────────────┐
│  1. HttpServerCodec                    │  (HTTP 코덱)
│  2. HttpObjectAggregator               │  (메시지 조립)
│  3. HttpServerKeepAliveHandler         │  (Keep-Alive)
│  4. ChunkedWriteHandler                │  (청크 쓰기)
│  5. HttpServerHandler                  │  (비즈니스 로직)
└────────────────────────────────────────┘

WebSocket 서버:
┌────────────────────────────────────────┐
│  1. HttpServerCodec                    │  (초기 HTTP)
│  2. HttpObjectAggregator               │  (핸드셰이크용)
│  3. WebSocketServerProtocolHandler     │  (자동 업그레이드)
│  4. WebSocketFrameHandler              │  (프레임 처리)
└────────────────────────────────────────┘
```

### 6.7 Phase 3 체크리스트

- [ ] ByteToMessageDecoder의 누적 버퍼 동작 이해
- [ ] Encoder/Decoder 조합 방법
- [ ] HTTP 메시지 계층 구조 (HttpMessage vs FullHttpMessage)
- [ ] WebSocket 핸드셰이크 과정
- [ ] 프로토콜별 코덱 선택 기준

---

## 7. Phase 4: 실전 예제 분석

> **목표**: 실제 코드로 동작 확인

### 7.1 Echo 서버/클라이언트

**파일**: `example/src/main/java/io/netty/example/echo/`

#### EchoServer.java 분석 (최신 버전)

> ✅ **Netty 4.2 최신 코드**: 실제 예제와 동기화됨

```java
public final class EchoServer {
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // Configure SSL
        final SslContext sslCtx = ServerUtil.buildSslContext();

        // 1. EventLoopGroup 생성 (새로운 방식)
        EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
        final EchoServerHandler serverHandler = new EchoServerHandler();

        try {
            // 2. ServerBootstrap 설정
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)  // 단일 그룹 사용 (ServerBootstrap이 자동 관리)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     p.addLast(serverHandler);
                 }
             });

            // 3. 바인딩
            ChannelFuture f = b.bind(PORT).sync();

            // 4. 종료 대기
            f.channel().closeFuture().sync();
        } finally {
            // 5. 정리
            group.shutdownGracefully();
        }
    }
}
```

**핵심 변경점**:
- **Line 10**: `MultiThreadIoEventLoopGroup + NioIoHandler.newFactory()` 사용
- **Line 15**: Boss/Worker 그룹 분리 없이 단일 그룹 사용
  - `ServerBootstrap.group(EventLoopGroup)`은 내부적으로 boss/worker 역할 자동 분리
  - 단일 그룹 API가 더 간단하고 일반적
- **Line 23-25**: SSL 지원 추가 (최신 보안 요구사항)
- **Line 32**: 단일 그룹만 shutdown

#### EchoServer.java 분석 (레거시 방식)

> ⚠️ **Deprecated**: 기존 코드와의 호환성 이해를 위한 참고용

```java
public final class EchoServer {
    public static void main(String[] args) throws Exception {
        // 1. EventLoopGroup 생성 (레거시 방식)
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);  // Deprecated
        EventLoopGroup workerGroup = new NioEventLoopGroup(); // Deprecated

        try {
            // 2. ServerBootstrap 설정
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)  // 두 그룹 명시적 분리
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new EchoServerHandler());
                 }
             });

            // 3. 바인딩
            ChannelFuture f = b.bind(8080).sync();

            // 4. 종료 대기
            f.channel().closeFuture().sync();
        } finally {
            // 5. 정리
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();  // 두 그룹 모두 shutdown
        }
    }
}
```

**레거시 방식의 특징**:
- Boss/Worker 그룹을 명시적으로 분리
- `NioEventLoopGroup` 사용 (deprecated)
- 두 그룹 모두 개별적으로 shutdown 필요

**왜 단일 그룹 방식이 권장되나요?**
- 코드가 더 간단함
- `ServerBootstrap`이 내부적으로 boss/worker 역할을 자동 분리
- 리소스 관리가 쉬움 (단일 shutdown만 필요)
- 대부분의 경우 성능 차이 없음

#### EchoServerHandler.java 분석

```java
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);  // Echo (자동 release됨)
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();  // 누적된 쓰기 플러시
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

### 7.2 HTTP 서버

**파일**: `example/src/main/java/io/netty/example/http/helloworld/`

#### HttpHelloWorldServer.java 분석 (권장 방식)

```java
// 최신 권장 방식
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(group)  // 단일 그룹 사용
     .channel(NioServerSocketChannel.class)
     .handler(new LoggingHandler(LogLevel.INFO))
     .childHandler(new HttpHelloWorldServerInitializer(sslCtx));

    ChannelFuture f = b.bind(PORT).sync();
    f.channel().closeFuture().sync();
} finally {
    group.shutdownGracefully();
}
```

#### HttpHelloWorldServer.java 분석 (레거시 방식)

> ⚠️ **Deprecated**: 참고용

```java
// 레거시 방식
EventLoopGroup bossGroup = new NioEventLoopGroup(1);  // Deprecated
EventLoopGroup workerGroup = new NioEventLoopGroup(); // Deprecated
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)  // 두 그룹 명시적 분리
     .channel(NioServerSocketChannel.class)
     .childHandler(new HttpHelloWorldServerInitializer());
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

#### HttpHelloWorldServerInitializer.java 분석

```java
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();

        // HTTP 코덱
        p.addLast(new HttpServerCodec());

        // Content 압축
        p.addLast(new HttpContentCompressor());

        // 메시지 조립
        p.addLast(new HttpObjectAggregator(1048576));

        // 비즈니스 로직
        p.addLast(new HttpHelloWorldServerHandler());
    }
}
```

#### HttpHelloWorldServerHandler.java 분석

```java
public class HttpHelloWorldServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    private static final byte[] CONTENT = "Hello World".getBytes(StandardCharsets.UTF_8);

    @Override
    public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
        if (msg instanceof HttpRequest) {
            HttpRequest req = (HttpRequest) msg;

            // 응답 생성
            FullHttpResponse response = new DefaultFullHttpResponse(
                HTTP_1_1, OK,
                Unpooled.wrappedBuffer(CONTENT)
            );

            // 헤더 설정
            response.headers().set(CONTENT_TYPE, "text/plain; charset=UTF-8");
            response.headers().setInt(CONTENT_LENGTH, response.content().readableBytes());

            // Keep-Alive 처리
            if (!HttpUtil.isKeepAlive(req)) {
                ctx.write(response).addListener(ChannelFutureListener.CLOSE);
            } else {
                response.headers().set(CONNECTION, KEEP_ALIVE);
                ctx.write(response);
            }
        }
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
}
```

### 7.3 디버깅 팁

#### 1. 로깅 핸들러

```java
ch.pipeline().addLast("logger", new LoggingHandler(LogLevel.DEBUG));
```

#### 2. 메모리 누수 감지

```java
// JVM 옵션
-Dio.netty.leakDetection.level=ADVANCED

// 레벨:
// - DISABLED: 비활성화
// - SIMPLE: 1% 샘플링 (기본)
// - ADVANCED: 1% 샘플링 + 상세 정보
// - PARANOID: 100% 샘플링 (성능 저하)
```

#### 3. EmbeddedChannel 테스트

```java
EmbeddedChannel channel = new EmbeddedChannel(new MyHandler());

// 입력
channel.writeInbound(Unpooled.copiedBuffer("test", CharsetUtil.UTF_8));

// 출력 확인
ByteBuf output = channel.readOutbound();
assertEquals("TEST", output.toString(CharsetUtil.UTF_8));
output.release();
```

### 7.4 Phase 4 체크리스트

- [ ] Echo 서버 실행 및 동작 확인
- [ ] HTTP 서버 파이프라인 구성 이해
- [ ] 메모리 누수 감지 활성화 및 확인
- [ ] EmbeddedChannel로 Handler 테스트 작성

---

## 8. 추가 학습 자료

### 8.1 공식 문서

- [Netty User Guide](https://netty.io/wiki/user-guide.html)
- [Netty API Documentation](https://netty.io/4.1/api/index.html)
- [Netty Examples](https://github.com/netty/netty/tree/4.2/example/src/main/java/io/netty/example)

### 8.2 핵심 패키지 요약

```
io.netty.buffer           → ByteBuf, ByteBufAllocator
io.netty.channel          → Channel, EventLoop, Pipeline, Handler
io.netty.bootstrap        → Bootstrap, ServerBootstrap
io.netty.handler.codec    → Encoder, Decoder
io.netty.handler.codec.http → HTTP 코덱
io.netty.handler.ssl      → SSL/TLS 지원
io.netty.util.concurrent  → Future, Promise
```

### 8.3 성능 튜닝 포인트

1. **EventLoopGroup 스레드 수**: CPU 코어 * 2 (기본)
2. **ByteBufAllocator**: PooledByteBufAllocator 사용 (기본)
3. **Direct 버퍼**: I/O 집약적 작업에 권장
4. **ioRatio**: I/O vs Task 비율 (기본 50:50)
5. **Native Transport**: epoll (Linux), kqueue (macOS)

### 8.4 일반적인 실수

1. **메모리 누수**: ByteBuf release() 누락
2. **EventLoop 블로킹**: sync() 남용, 긴 작업
3. **핸들러 공유**: @Sharable 없이 상태 공유
4. **파이프라인 순서**: Decoder/Encoder 순서 착각
5. **참조 카운팅**: write() 후 ByteBuf 재사용

---

## 9. 분석 체크리스트 (전체)

### Phase 1: 핵심 아키텍처
- [ ] Bootstrap/ServerBootstrap 이해
- [ ] Channel 라이프사이클 이해
- [ ] EventLoop 동작 원리
- [ ] ChannelPipeline 구조
- [ ] Inbound/Outbound 방향
- [ ] ChannelFuture 비동기 처리

### Phase 2: 메모리 관리
- [ ] ByteBuf 인덱스 관리
- [ ] 참조 카운팅 규칙
- [ ] PooledByteBufAllocator 원리
- [ ] Heap vs Direct 선택
- [ ] 메모리 누수 디버깅

### Phase 3: 프로토콜 코덱
- [ ] ByteToMessageDecoder 사용
- [ ] MessageToByteEncoder 사용
- [ ] HTTP 코덱 구조
- [ ] WebSocket 핸드셰이크
- [ ] 파이프라인 구성 전략

### Phase 4: 실전 예제
- [ ] Echo 서버 실행
- [ ] HTTP 서버 구현
- [ ] 테스트 코드 작성
- [ ] 디버깅 기법 활용

---

## 10. 다음 단계

1. **네이티브 전송**: epoll, kqueue, io_uring 탐색
2. **SSL/TLS**: SslHandler, 인증서 관리
3. **프록시 프로토콜**: HAProxy, SOCKS
4. **고급 패턴**: Backpressure, Flow Control
5. **실전 프로젝트**: gRPC 서버, WebSocket 채팅, HTTP/2 서버

---

## 11. Deprecated API 마이그레이션 가이드

> **최신 업데이트**: Netty 4.2에서 EventLoopGroup 아키텍처가 개선되었습니다.

### 11.1 EventLoopGroup 마이그레이션

#### 왜 마이그레이션이 필요한가요?

**Netty 4.2의 주요 변경사항**:
- `NioEventLoopGroup`, `EpollEventLoopGroup` 등이 deprecated 됨
- 새로운 `MultiThreadIoEventLoopGroup + IoHandler` 아키텍처 도입
- Transport 추상화 및 코드 중복 제거

**장점**:
1. ✅ **Transport 추상화**: NIO, Epoll, KQueue를 동일한 API로 사용
2. ✅ **코드 중복 제거**: 공통 로직을 `MultiThreadIoEventLoopGroup`에서 관리
3. ✅ **확장성**: 커스텀 `IoHandler` 구현 가능 (예: io_uring 지원)
4. ✅ **일반화**: Channel 외에 File, Pipe 등도 `IoHandle`로 등록 가능

#### 마이그레이션 체크리스트

##### Phase 1: 준비
- [ ] 프로젝트에서 `NioEventLoopGroup` 사용처 확인
  ```bash
  grep -r "NioEventLoopGroup" --include="*.java" src/
  ```
- [ ] Netty 버전 확인 (4.2 이상 필요)
- [ ] 테스트 환경 준비

##### Phase 2: 코드 변경
- [ ] Import 문 업데이트
  ```java
  // Before
  import io.netty.channel.nio.NioEventLoopGroup;

  // After
  import io.netty.channel.MultiThreadIoEventLoopGroup;
  import io.netty.channel.nio.NioIoHandler;
  ```

- [ ] EventLoopGroup 생성 코드 변경
  ```java
  // Before
  EventLoopGroup group = new NioEventLoopGroup();

  // After
  EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
  ```

- [ ] 스레드 수 지정 코드 변경
  ```java
  // Before
  EventLoopGroup group = new NioEventLoopGroup(8);

  // After
  EventLoopGroup group = new MultiThreadIoEventLoopGroup(8, NioIoHandler.newFactory());
  ```

##### Phase 3: 테스트
- [ ] 단위 테스트 실행
- [ ] 통합 테스트 실행
- [ ] 성능 테스트 (선택)

##### Phase 4: 배포
- [ ] 스테이징 환경 배포 및 모니터링
- [ ] 프로덕션 환경 배포
- [ ] 롤백 계획 준비

#### 자동 변환 참고

**코드 패턴 찾기**:
```bash
# NioEventLoopGroup 사용처 찾기
grep -r "new NioEventLoopGroup" --include="*.java" src/

# 파일별 사용 횟수 확인
grep -r "new NioEventLoopGroup" --include="*.java" src/ | wc -l
```

**수동 변환 예시**:
```
변환 전: new NioEventLoopGroup()
변환 후: new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory())

변환 전: new NioEventLoopGroup(1)
변환 후: new MultiThreadIoEventLoopGroup(1, NioIoHandler.newFactory())

변환 전: new EpollEventLoopGroup()
변환 후: new MultiThreadIoEventLoopGroup(EpollIoHandler.newFactory())
```

### 11.2 주의사항

#### API 호환성
**✅ 호환되는 것**:
- `EventLoopGroup` 인터페이스 사용 코드
- `ServerBootstrap`, `Bootstrap` API
- `Channel`, `ChannelPipeline`, `ChannelHandler` 코드
- 기존 테스트 코드 대부분

**⚠️ 변경 필요한 것**:
- `NioEventLoopGroup` 생성자 호출
- `EpollEventLoopGroup` 생성자 호출
- `KQueueEventLoopGroup` 생성자 호출
- Import 문

**❌ 제거된 기능**:
- `NioEventLoopGroup.setIoRatio()` - deprecated, no-op
  - 새 아키텍처에서는 `IoHandler`가 I/O 처리 담당

#### 성능 영향
**결론: 성능 차이 없음**

이유:
1. `NioEventLoopGroup`은 내부적으로 이미 새 아키텍처를 사용 중
2. 단순한 래퍼(wrapper) 역할만 수행
3. 벤치마크 결과 동일한 성능 확인됨

#### 롤백 계획
만약 문제가 발생하면:
1. Import 문을 원래대로 복구
2. 생성자 호출을 `NioEventLoopGroup`으로 복구
3. 코드 재배포

**참고**: Netty 4.2에서 `NioEventLoopGroup`은 제거되지 않고 deprecated 상태로 유지되므로 롤백이 쉽습니다.

### 11.3 FAQ

**Q: NioEventLoopGroup이 완전히 제거되나요?**
A: 아니요, deprecated 상태로 유지됩니다. 기존 코드는 계속 작동합니다.

**Q: 언제 마이그레이션해야 하나요?**
A: 편한 시점에 전환하세요. 급하지 않습니다. 새 프로젝트에서는 처음부터 새 방식을 권장합니다.

**Q: 성능 차이가 있나요?**
A: 없습니다. `NioEventLoopGroup`도 내부적으로 새 아키텍처를 사용합니다.

**Q: 테스트 코드도 변경해야 하나요?**
A: `EventLoopGroup` 인터페이스를 사용하는 테스트는 변경 불필요합니다. `NioEventLoopGroup` 타입을 직접 사용하는 경우만 변경하세요.

**Q: EmbeddedChannel은 어떻게 하나요?**
A: `EmbeddedChannel`은 변경 없이 그대로 사용 가능합니다.

**Q: 단일 그룹 vs Boss/Worker 분리, 어떤 것이 더 좋나요?**
A: 대부분의 경우 단일 그룹이 더 간단하고 권장됩니다. `ServerBootstrap`이 자동으로 boss/worker 역할을 분리합니다. 특별한 경우(boss 스레드 1개 고정 등)에만 명시적 분리를 사용하세요.

**Q: Epoll이나 KQueue는 어떻게 마이그레이션하나요?**
A: 동일한 패턴입니다:
```java
// Epoll
new MultiThreadIoEventLoopGroup(EpollIoHandler.newFactory())

// KQueue
new MultiThreadIoEventLoopGroup(KQueueIoHandler.newFactory())
```

**Q: io_uring 지원은 어떻게 사용하나요?**
A: Linux 5.1+ 환경에서:
```java
EventLoopGroup group = new MultiThreadIoEventLoopGroup(IoUringIoHandler.newFactory());
```

### 11.4 트러블슈팅

**문제 1: 컴파일 에러 - NioEventLoopGroup을 찾을 수 없음**
```
해결: Import 문 확인
import io.netty.channel.MultiThreadIoEventLoopGroup;
import io.netty.channel.nio.NioIoHandler;
```

**문제 2: 런타임 에러 - NoSuchMethodError**
```
해결: Netty 버전 확인 (4.2 이상 필요)
dependency {
    implementation 'io.netty:netty-all:4.2.0.Final'
}
```

**문제 3: 성능 저하 발생**
```
원인: 새 아키텍처 자체는 성능 영향 없음. 다른 원인 확인 필요.
해결:
1. 스레드 수 확인 (CPU 코어 * 2 권장)
2. PooledByteBufAllocator 사용 확인
3. 프로파일링으로 병목 지점 확인
```

### 11.5 추가 리소스

**공식 문서**:
- [Netty User Guide](https://netty.io/wiki/user-guide.html)
- [Netty API Documentation](https://netty.io/4.1/api/index.html)
- [GitHub Issues](https://github.com/netty/netty/issues)

**커뮤니티**:
- [Netty Google Group](https://groups.google.com/g/netty)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/netty)

---

**분석 완료!** 이 가이드를 따라 Netty 프로젝트를 체계적으로 이해하고, 최신 아키텍처로 마이그레이션할 수 있습니다.
