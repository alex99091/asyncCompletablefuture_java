## CompletableFuture를 이용한 병렬처리 리팩토링 사례

## 🔍 적용 배경

* 기존 코드에서는 하나의 업무 요청 흐름에 대해 4\~5개의 처리 메서드가 **완전히 순차적으로 실행**되고 있었고,
* 각 메서드는 개별 I/O 기반 작업(DB 조회, 외부 인증, 계약정보 호출 등)으로 수 초의 시간이 소요됐습니다.
* 이 구조를 유지하면서도 **동시에 실행 가능한 작업을 병렬화**하여 응답 시간을 단축하는 것이 목표였습니다.

---

## 🧾 기존 순차 처리 방식 (예시 코드 - 설명용)

```java
public BizResponse processData(BizRequest request) throws BizException {
    BizResponse response = new BizResponse();

    // 1. 사용자 지역 정보 조회
    this.loadRegionInfo(request);

    // 2. 본인 확인 정보 조회
    this.verifyIdentityInfo(request);

    // 3. 사용자 프로필 조회
    this.fetchUserProfile(request);

    // 4. 계약 관련 정보 조회
    this.loadContractInfo(request);

    return response;
}
```

* 모든 처리가 **완료되어야 다음 단계로 넘어가기 때문에 병목**이 발생.

---

## 🚀 CompletableFuture로 병렬 처리 (리팩토링 예시)

```java
public BizResponse processDataAsync(BizRequest request) throws BizException {
    BizResponse response = new BizResponse();

    // 병렬로 실행 가능한 각 Task 정의 (예외 래핑 포함)
    CompletableFuture<Void> regionTask = CompletableFuture.runAsync(() -> safeRun(() -> loadRegionInfo(request)));
    // runAsync(): 별도 쓰레드에서 Runnable 실행, 반환값 없음
    // safeRun(): 내부 예외를 CompletionException으로 래핑하여 처리

    CompletableFuture<Void> identityTask = CompletableFuture.runAsync(() -> safeRun(() -> verifyIdentityInfo(request)));
    // verifyIdentityInfo() 메서드를 병렬 비동기 작업으로 실행

    CompletableFuture<Void> profileTask = CompletableFuture.runAsync(() -> safeRun(() -> fetchUserProfile(request)));
    // fetchUserProfile(): 사용자 정보를 외부 API에서 가져오는 작업을 병렬 처리

    CompletableFuture<Void> contractTask = CompletableFuture.runAsync(() -> safeRun(() -> loadContractInfo(request)));
    // loadContractInfo(): 계약정보를 별도 비동기 쓰레드에서 가져오기

    // 모든 작업이 완료될 때까지 대기 (예외 발생 시 CompletionException으로 래핑됨)
    try {
        CompletableFuture.allOf(regionTask, identityTask, profileTask, contractTask).join();
        // allOf(): 여러 CompletableFuture들을 병렬 실행 후 모두 완료되기를 기다림
        // join(): 예외가 발생하면 CompletionException으로 래핑되어 즉시 전파됨
    } catch (CompletionException ce) {
        throw new BizException("비동기 병렬 처리 중 오류 발생", ce);
    }

    return response;
}

// 람다 내 예외 처리를 위한 공통 유틸 함수
private void safeRun(Runnable action) {
    try {
        action.run();
    } catch (Exception e) {
        throw new CompletionException(e);
        // CompletionException: CompletableFuture 내부에서 발생한 예외를 감싸기 위한 표준 예외 타입
    }
}
```

---

## ✨ 핵심 메서드 요약 (Java 표준 API 기반)

| 메서드                            | 설명                                                              |
| ------------------------------ | --------------------------------------------------------------- |
| `runAsync(Runnable)`           | 반환값 없이 백그라운드 작업 실행. 기본적으로 ForkJoinPool 공용 쓰레드 사용                |
| `allOf(...)`                   | 여러 CompletableFuture를 배열로 받아 모두 완료될 때까지 대기                      |
| `join()`                       | 해당 CompletableFuture가 끝날 때까지 대기, 예외 발생 시 CompletionException 발생 |
| `CompletionException`          | 비동기 내부 예외를 감싸기 위한 예외. 외부에서 catch 가능                             |
| `thenCompose()`                | 결과값을 다른 CompletableFuture로 연결. 의존성 있는 비동기 로직에서 사용               |
| `handle()` / `exceptionally()` | 예외 발생 시 후속 처리 로직 연결 가능                                          |

---

## 🎯 적용 효과 및 기대 이점

| 항목    | 효과                         |
| ----- |----------------------------|
| 처리 시간 | 기존 15초 이상 → 평균 4~5초로 단축    |
| 병목 제거 | 순차 구조 → 병렬 실행으로 CPU 사용률 향상 |
| 확장성   | 추가 작업을 Task로 쉽게 분리 가능      |
| 유지보수  | 공통 예외 처리 유틸을 도입해 코드 중복 제거  |

---

## 🧠 사내 적용 시 유의사항

* 모든 로직이 병렬화에 적합한지 반드시 검토 (예: DB 락, 순서 의존성 등)
* 공통 ThreadPool 또는 별도 Executor 설정 필요시 직접 지정 가능
* 실패 허용이 필요한 경우 `handle`, `exceptionally`, `whenComplete` 등 활용 가능

---

> ✅ 위 코드는 실제 업무 코드에서 추상화한 예시입니다.
> “이런 방식으로 병렬화하면 유사 구조에서 성능 개선과 구조 개선이 동시에 가능합니다”라는 관점에서 설계된 리팩토링 예제입니다.
