---
layout: post
title: "SSE 실시간 알림 Production 장애 해결"
date: 2025-06-27
categories: backend
---

# SSE 실시간 알림 시스템 Production 장애 해결

프랜차이즈 주문 및 결재 관리 시스템을 개발하면서  
SSE(Server-Sent Events)를 활용한 실시간 알림 시스템을 구현했습니다.

하지만 Local 환경에서는 정상 동작하던 알림이  
Production 환경에서는 실시간으로 전달되지 않는 문제가 발생했습니다.

이 글에서는 해당 문제의 원인과 해결 과정을 정리합니다.

---

# 문제 상황

알림 시스템은 다음 구조로 동작합니다.

1. 비즈니스 이벤트 발생  
2. Notification 생성  
3. SSE 연결을 통해 사용자에게 이벤트 Push  

하지만 실제 운영 환경에서는 다음 문제가 발생했습니다.

Local 환경

- 알림 이벤트 발생
- 즉시 실시간 알림 수신

Production 환경

- 알림 이벤트 발생
- 클라이언트에서 알림 수신되지 않음
- 페이지 새로고침 시 알림 확인 가능

즉 SSE 연결은 유지되고 있었지만 실제 이벤트 Push가 정상적으로 동작하지 않는 상태였습니다.

---

# 시스템 구조

알림 시스템은 다음과 같이 구성되어 있습니다.

Business Event  
↓  
Notification Service  
↓  
Notification 저장  
↓  
EmitterRepository  
↓  
SseEmitter → Client

사용자별 SSE 연결은 EmitterRepository에서 관리합니다.

---

# 원인 분석

문제를 분석하는 과정에서 다음 세 가지 원인을 확인했습니다.

---

## 1. Transaction 내부에서 SSE 이벤트 전송

기존 코드에서는 알림 생성과 SSE 이벤트 전송이 동일한 트랜잭션 내부에서 실행되고 있었습니다.

```java
@Transactional
public void send(UserEntity receiver, NotificationType notificationType, NotificationTarget target) {

    Notification notification = notificationRepository.save(
        createNotification(receiver, notificationType, target)
    );

    Map<String, SseEmitter> emitters =
        emitterRepository.findAllEmitterStartWithByUserId(receiverIdStr);

    emitters.forEach((key, emitter) -> {
        sendNotification(emitter, eventId, key, response);
    });
}
````

이 구조에서는 다음과 같은 문제가 발생할 수 있습니다.

* DB Transaction commit 이전에 SSE 이벤트 전송
* Production 환경에서 Thread blocking 발생
* 이벤트 전송 실패 가능성

---

## 2. ConcurrentModificationException 가능성

Emitter Map을 순회하면서 삭제가 발생할 가능성이 있었습니다.

```java
emitters.forEach((key, emitter) -> {
    emitterRepository.deleteById(key);
});
```

Map을 순회하면서 구조가 변경되면 ConcurrentModificationException이 발생할 수 있습니다.

---

## 3. 끊어진 SSE 연결 관리 문제

Heartbeat 로직에서 끊어진 연결이 제대로 제거되지 않으면

* emitter 객체가 계속 Repository에 남음
* 장기적으로 메모리 누수 발생 가능

문제가 발생할 수 있습니다.

---

# 해결 방법

문제를 해결하기 위해 알림 시스템 구조를 다음과 같이 개선했습니다.

---

## 1. SSE 전송을 Async 처리로 분리

알림 생성과 SSE 전송을 분리하여 Transaction 완료 이후에 SSE 이벤트가 전송되도록 수정했습니다.

```java
@Transactional
public void send(UserEntity receiver, NotificationType notificationType, NotificationTarget target) {

    Notification notification = notificationRepository.save(
        createNotification(receiver, notificationType, target)
    );

    sendSseNotificationAsync(receiverIdStr, eventId, notification);
}

@Async
public void sendSseNotificationAsync(String receiverIdStr, String eventId, Notification notification) {

    Map<String, SseEmitter> emitters =
        emitterRepository.findAllEmitterStartWithByUserId(receiverIdStr);

    emitters.forEach((key, emitter) -> {
        sendNotification(emitter, eventId, key, response);
    });
}
```

이 구조로 변경하면서

* Transaction commit 이후 SSE 이벤트 전송
* Thread blocking 문제 해결
* Production 환경에서도 안정적인 이벤트 전달

이 가능해졌습니다.

---

## 2. ConcurrentModificationException 방지

Emitter Map을 복사한 뒤 순회하도록 변경했습니다.

```java
Map<String, SseEmitter> emittersCopy = new HashMap<>(emitters);

emittersCopy.forEach((key, emitter) -> {
    sendNotification(emitter, eventId, key, response);
});
```

---

## 3. Heartbeat 기반 연결 관리

30초마다 heartbeat 이벤트를 전송하여 끊어진 SSE 연결을 감지하고 emitter를 정리하도록 구현했습니다.

```java
@Scheduled(fixedRate = 30000)
public void sendHeartbeat() {

    Map<String, SseEmitter> emitters =
        emitterRepository.findAllEmitterStartWithByUserId("");

    emitters.forEach((emitterId, emitter) -> {
        try {
            emitter.send(
                SseEmitter.event()
                    .name("heartbeat")
                    .data("ping")
            );
        } catch (IOException e) {
            emitterRepository.deleteById(emitterId);
        }
    });
}
```

---

# 결과

구조 개선 이후 Production 환경에서도

* 알림 이벤트 발생 시
* 클라이언트에서 즉시 실시간 알림 수신

이 가능해졌습니다.

또한 다음과 같은 개선 효과를 얻을 수 있었습니다.

* SSE 연결 안정성 향상
* 끊어진 연결 정리
* emitter 메모리 누수 방지
* Production 환경 실시간 알림 정상 동작

---

# 배운 점

이번 문제 해결을 통해 다음과 같은 점을 배울 수 있었습니다.

* 실시간 이벤트 시스템에서 Transaction과 Async 처리 분리의 중요성
* SSE 연결 관리에서 Heartbeat 기반 연결 유지 전략
* Concurrent 환경에서 발생할 수 있는 Collection 수정 문제
* Local 환경과 Production 환경의 동작 차이에 대한 이해

```