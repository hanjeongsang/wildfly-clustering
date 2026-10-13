# ⚡ WildFly Clustering 10.0.8.Final - 세션 배치 충돌 패치 버전

이 저장소는 **WildFly 40.0.1.Final** 환경에서 발생하는 세션 배치 충돌 예외를 해결하기 위해, 최신 `main` 브랜치에 이미 공식 적용된 픽스([ceae79b](https://github.com/wildfly-clustering/wildfly-clustering/commit/ceae79bbb2e36e0b1117ad38dcf4f5a2e16388bd))의 핵심 수정 사항을 하위 버전인 **`10.0.8.Final`** 코드베이스에서도 동일하게 정상 동작할 수 있도록 백포팅(Backporting)하여 통합한 패치 버전입니다.

상위 버전의 WAS 엔진으로 전체 업그레이드를 진행하기 어려운 레거시 또는 고정 버전 환경에서, **단 1개의 모듈 JAR 파일 교체만으로 시스템 안정성을 유지하기 위한 목적**으로 관리됩니다.

이 소스는 공식자료가 아니며 개인적인 기록/참고용 자료 입니다.

---

## 🚨 이슈 상세: 세션 마감(close)과 재개(resume) 경합 버그

동일한 세션 쿠키를 사용하는 비동기 요청이나 브라우저 더블 클릭(연타) 등이 나노초 단위로 동시에 다수 유입될 때 발생합니다.

1. 선행 HTTP 요청 스레드가 작업을 마치고 최종적으로 응답을 처리하는 과정에서 **세션을 캐시와 동기화하고 닫는 단계(`CachedSession.close()`)**에 진입합니다.
2. 이 짧은 정리 찰나(수 ms)에, 뒤따라 유입된 후속 요청 스레드가 동일한 세션 ID에 접근하여 **세션 배치를 활성화(`resume()`)**하려 시도합니다.
3. WildFly의 분산 세션 관리 메커니즘(`ThreadContextBatch`)은 이전 배치가 완전히 닫히지 않은 상태에서 새로운 배치가 재개되는 것을 감지하고, 정합성을 보호하기 위해 즉시 **`IllegalStateException` 및 `ContextualException`**을 유발하여 연쇄적인 요청 실패와 WAS 마비를 일으킵니다.

---

## 📋 정규화된 현장 에러 로그 (Stack Trace)

```text
ERROR [io.undertow.request] (default task-X) UT005023: Exception handling request to /sso/login: java.util.concurrent.CompletionException: java.lang.IllegalStateException: org.wildfly.clustering.cache.infinispan.batch.ThreadContextBatch$2@XXXXXXXX
    at java.base/java.util.concurrent.CompletableFuture.encodeThrowable(CompletableFuture.java:315)
    at java.base/java.util.concurrent.CompletableFuture.completeThrowable(CompletableFuture.java:320)
    at java.base/java.util.concurrent.CompletableFuture$UniApply.tryFire(CompletableFuture.java:646)
    ...
    at org.infinispan.core@16.0.13//org.infinispan.remoting.transport.jgroups.JGroupsTransport.processMessage(JGroupsTransport.java:1470)
    at org.jgroups@5.5.5.Final//org.jgroups.util.SubmitToThreadPool$SingleMessageHandler.run(SubmitToThreadPool.java:99)
    at java.base/java.lang.Thread.run(Thread.java:1583)
Caused by: java.lang.IllegalStateException: org.wildfly.clustering.cache.infinispan.batch.ThreadContextBatch$2@XXXXXXXX
    at org.wildfly.clustering.cache.infinispan.common@10.0.8.Final//org.wildfly.clustering.cache.infinispan.batch.ThreadContextBatch$2.resume(ThreadContextBatch.java:82)
    at org.wildfly.clustering.session.cache@10.0.8.Final//org.wildfly.clustering.session.cache.CachedSession.resume(CachedSession.java:47)
    at org.wildfly.clustering.session.cache@10.0.8.Final//org.wildfly.clustering.session.cache.CachedSessionManager$1.accept(CachedSessionManager.java:66)
    at org.wildfly.clustering.session.cache@10.0.8.Final//org.wildfly.clustering.session.cache.CachedSessionManager$1.accept(CachedSessionManager.java:60)
    at org.wildfly.clustering.server.spi@10.0.8.Final//org.wildfly.clustering.server.cache.CacheStrategy$2$1.remove(CacheStrategy.java:155)
    at org.wildfly.clustering.session.cache@10.0.8.Final//org.wildfly.clustering.session.cache.CachedSession.close(CachedSession.java:42)
    ... 43 more
Caused by: org.wildfly.clustering.cache.infinispan.batch.ContextualException: XX
    ... 54 more
```

---

## 🛠️ 패치 방법

본 저장소의 `patch-10.0.8.Final` 브랜치는 **최신 `main` 브랜치에 이미 적용되어 검증된 공식 수정 내역(`resumeWithContext()`를 통한 자동 트랜잭션 컨텍스트 보존 로직)을 하위 `10.0.8.Final` 버전 소스에 백포팅(Backporting)하여 합친 것**입니다.

다음 중 편한 방법을 선택하여 WildFly 서버에 반영할 수 있습니다.

### 방법 A: 소스 코드 직접 빌드하여 적용하기

1. **소스를 클론하고 패치 브랜치로 이동합니다:**
   ```bash
   git clone https://github.com/hanjeongsang/wildfly-clustering.git
   cd wildfly-clustering
   git checkout patch-10.0.8.Final
   ```

2. **Maven을 이용하여 수정한 세션 캐시 모듈만 타겟팅하여 패키징합니다:**
   ```bash
   mvn clean package -DskipTests -pl session/cache -am
   ```

3. **빌드 완료 후 생성된 타겟 패치 모듈을 확인합니다:**
   * 패치 결과물 생성 경로:  
     `./session/cache/target/wildfly-clustering-session-cache-10.0.8.Final.jar`

4. **빌드된 패치 JAR 파일을 WildFly 모듈 경로로 복사하여 원본 파일과 전격 교체합니다:**
   ```bash
   # (주의) {your_wildfly_home} 부분은 실제 WildFly 설치 디렉터리 경로로 지정해 주세요.
   cp -rp ./session/cache/target/wildfly-clustering-session-cache-10.0.8.Final.jar /{your_wildfly_home}/modules/system/layers/base/org/wildfly/clustering/session/cache/main/wildfly-clustering-session-cache-10.0.8.Final.jar
   ```

5. **적용 완료 후 WildFly 클러스터 노드들을 재기동합니다.**

---

### 방법 B: 미리 컴파일된 패치 완료 JAR 다운로드 및 적용

로컬 환경에 Maven 빌드 환경 구축이 번거로운 경우, 사전에 빌드 완료된 완제품 JAR 파일을 릴리즈에서 받아 즉시 적용할 수 있습니다.

1. 우측의 **[Releases]** 탭으로 이동하여 `wildfly-clustering-session-cache-10.0.8.Final.jar` 파일을 다운로드합니다.
2. WildFly 서버 내 해당 모듈 경로의 원본 파일을 백업합니다:
   ```bash
   mv /{your_wildfly_home}/modules/system/layers/base/org/wildfly/clustering/session/cache/main/wildfly-clustering-session-cache-10.0.8.Final.jar /{your_wildfly_home}/modules/system/layers/base/org/wildfly/clustering/session/cache/main/wildfly-clustering-session-cache-10.0.8.Final.jar.orig
   ```
3. 다운로드한 패치 JAR 파일을 해당 위치로 복사합니다:
   ```bash
   cp /path/to/downloaded/wildfly-clustering-session-cache-10.0.8.Final.jar /{your_wildfly_home}/modules/system/layers/base/org/wildfly/clustering/session/cache/main/wildfly-clustering-session-cache-10.0.8.Final.jar
   ```
4. 적용 완료 후 **WildFly 클러스터 노드들을 재기동**합니다.
