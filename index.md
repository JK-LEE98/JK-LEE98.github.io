# 이준규 | Backend Developer

Spring Boot 기반 백엔드 개발자입니다.  
DDD 기반 서비스 설계와 실시간 알림 시스템 구현 경험이 있습니다.

---

# Tech Stack

### Backend
- Java
- Spring Boot
- Spring Security
- JPA
- MyBatis

### Architecture
- DDD
- CQRS

### Infra
- AWS EC2

---

# Project

## Frans - 프랜차이즈 주문 관리 시스템

본사와 가맹점 간 주문 및 결재 프로세스를 관리하는 서비스입니다.

### Tech
Spring Boot / Spring Security / MyBatis / JPA / AWS EC2 / SSE

### 주요 기능

#### 주문 상태 관리

주문 상태 변경 시 Domain 객체에서 상태를 관리하고  
상태 변경이 발생하면 Notification Service를 통해 알림을 전송합니다.

#### 실시간 알림 시스템

SSE(Server-Sent Events)를 활용하여  
주문 상태 변경 및 결재 요청 알림을 실시간으로 전달합니다.

#### CQRS 구조

Command는 JPA  
Query는 MyBatis로 분리하여 설계했습니다.

---

# GitHub

프로젝트 Repository  
👉 https://github.com/본인프로젝트링크
