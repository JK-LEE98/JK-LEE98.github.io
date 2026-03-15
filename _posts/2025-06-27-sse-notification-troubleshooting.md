---
title: "SSE 실시간 알림 시스템 Production 장애 해결"
date: 2025-06-27
categories: [Backend, SSE, TroubleShooting]
tags: [SpringBoot, SSE, Async, Transaction]
---

# SSE 실시간 알림 Production 환경 문제 해결

프랜차이즈 주문 및 결재 관리 시스템을 개발하면서  
SSE(Server-Sent Events)를 활용한 실시간 알림 시스템을 구현했습니다.

알림은 주문 요청, 주문 상태 변경과 같은 이벤트가 발생했을 때  
사용자에게 실시간으로 전달되도록 설계했습니다.

하지만 실제 배포 환경에서 예상하지 못한 문제가 발생했습니다.

---

# 문제 상황

알림 시스템은 다음과 같은 구조로 동작합니다.

```

Client
│
│ subscribe()
▼
SseEmitter 생성
│
▼
EmitterRepository 저장
│
▼
알림 발생 시 emitter로 push

```

Local 환경에서는 실시간 알림이 정상적으로 동작했습니다.

```

Local 환경 → 실시간 알림 정상 동작

```

하지만 Production 환경에서는 다음과 같은 문제가 발생했습니다.

```

Production 환경 → 알림이 즉시 오지 않음
새로고침 이후 알림 수신

````

즉, SSE 연결은 유지되고 있었지만 **이벤트 push가 정상적으로 전달되지 않는 상태**였습니다.

---

# 원인 분석

코드를 분석하는 과정에서 SSE emitter 관리 과정에서  
여러 안정성 문제가 존재한다는 것을 확인했습니다.

---

# 1. Map 순회 중 수정 가능성

기존 코드에서는 emitter Map을 순회하면서 이벤트를 전송하고 있었습니다.

```java
emitters.forEach((key, emitter) -> {
    emitterRepository.saveEventCache(key, notification);
    sendNotification(emitter, eventId, key, ...);
});
````

하지만 이벤트 전송 중 emitter 연결이 끊어지면 다음 코드가 실행됩니다.

```java
emitterRepository.deleteById(key);
```

이 경우

```
Map 순회 중 구조 변경 발생
→ ConcurrentModificationException 가능
```

즉, **컬렉션을 순회하면서 동시에 수정이 발생할 가능성이 있었습니다.**

---

# 2. 끊어진 SSE 연결 정리 문제

SSE 연결이 끊어졌을 때 emitter가 제대로 제거되지 않으면
Repository에 끊어진 emitter가 계속 남을 수 있습니다.

이 경우

```
실제 연결이 없는 emitter에 이벤트 전송 시도
→ 예외 발생
→ 알림 전송 실패
```

문제가 발생할 수 있습니다.

---

# 3. Event Cache 삭제 중 수정 문제

Event Cache를 정리하는 과정에서도 동일한 문제가 존재했습니다.

기존 코드

```java
for (Map.Entry<String, Object> entry : eventCaches.entrySet()) {
    eventCache.remove(entry.getKey());
}
```

이 경우

```
Map 순회 중 삭제 발생
→ ConcurrentModificationException 가능
```

문제가 존재했습니다.

---

# 해결 방법

문제를 해결하기 위해 emitter 관리 구조를 다음과 같이 수정했습니다.

---

# 1. emitter Map 복사 후 순회

Map을 직접 순회하지 않고 복사본을 생성한 뒤 순회하도록 수정했습니다.

```java
Map<String, SseEmitter> emittersCopy = new HashMap<>(emitters);

emittersCopy.forEach((key, emitter) -> {
    sendNotification(emitter, eventId, key, response);
});
```

이를 통해 **Map 순회 중 수정 문제를 방지할 수 있었습니다.**

---

# 2. 끊어진 emitter 즉시 제거

이벤트 전송 중 예외가 발생하면 해당 emitter를 즉시 제거하도록 처리했습니다.

```java
catch (IOException e) {
    emitterRepository.deleteById(emitterId);
}
```

이를 통해 **끊어진 SSE 연결을 안정적으로 정리할 수 있도록 개선했습니다.**

---

# 3. Event Cache 삭제 방식 개선

삭제할 key를 먼저 수집한 뒤 삭제하도록 수정했습니다.

```java
List<String> keysToRemove = new ArrayList<>();

for (Map.Entry<String, Object> entry : eventCaches.entrySet()) {
    keysToRemove.add(entry.getKey());
}

keysToRemove.forEach(eventCache::remove);
```

이를 통해 **ConcurrentModificationException 발생 가능성을 제거했습니다.**

---

# 결과

구조 개선 이후 Production 환경에서도

```
알림 이벤트 발생
→ 클라이언트에서 즉시 실시간 알림 수신
```

이 정상적으로 동작하는 것을 확인했습니다.

또한 다음과 같은 개선 효과를 얻을 수 있었습니다.

* SSE 연결 안정성 향상
* 끊어진 emitter 자동 정리
* ConcurrentModificationException 방지
* Production 환경 실시간 알림 정상 동작

---

# 배운 점

이번 문제 해결을 통해 다음과 같은 점을 배울 수 있었습니다.

* SSE 시스템에서 emitter 관리의 중요성
* 컬렉션 순회 중 수정이 발생할 수 있는 동시성 문제
* 실시간 이벤트 시스템에서 안정적인 연결 관리 전략
* Local 환경과 Production 환경의 동작 차이
