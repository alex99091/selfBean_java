# selfBean_java

[![Java](https://img.shields.io/badge/Java-%23ED8B00.svg?style=for-the-badge&logo=java&logoColor=white)](https://www.java.com/)
[![Spring](https://img.shields.io/badge/Spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white)](https://spring.io/)

> **Note**: 본 프로젝트는 현대카드 H-Framework+ 버전에 관한 설명이며 실제 코드는 공개하지 않습니다.  
> 기술 구조 및 역할 위주로 정리된 문서 기반 포트폴리오입니다.

---

## 🧩 Overview

본 프로젝트는 Spring Framework와 AOP(Aspect-Oriented Programming)를 활용한  
`Self-Bean` 주입 문제 해결을 다룬 사례입니다.  
`Self-Bean` 주입을 통해 프록시를 적용하여 AOP 기능이 정상적으로 작동하도록 수정하였습니다.

- **적용 대상 시스템**: Spring 기반 서비스
- **주요 문제점**: AOP 기능이 프록시를 거치지 않고 직접 호출되며 발생한 Bean 호출 에러
- **해결책**: Self-Bean 주입을 통해 AOP 프록시 적용

---

## ❗ Problem

Spring의 AOP 기능을 사용하여 트랜잭션이나 비동기 처리와 같은 기능을 적용하려 했지만,  
내부 메서드 호출 시 프록시가 적용되지 않아 AOP가 정상적으로 동작하지 않는 문제가 발생하였습니다.  
이 문제를 해결하기 위해 `Self-Bean` 주입 방식이 적용되었습니다.

### 주요 이슈

1. **프록시 미적용**
   - Spring에서 AOP는 기본적으로 프록시 객체를 사용하여 메서드 호출을 가로채지만,  
   내부에서 직접 호출 시 프록시가 적용되지 않아 AOP 기능이 동작하지 않음.

2. **AOP 기능 미작동**
   - `@Transactional`이나 `@Async` 어노테이션을 사용했음에도 불구하고,  
   메서드가 직접 호출되면서 트랜잭션 처리나 비동기 처리 기능이 정상적으로 동작하지 않음.

---

## 🔎 As-Is 구조

기존 구조는 Spring에서 AOP 기능을 사용하면서도 내부 메서드 호출 시 프록시를 거치지 않아서 AOP가 적용되지 않았습니다.  
기존 구조는 다음과 같았습니다:

```java
@BXMApplication
public class Test {
    private DataBase dataBase;

    public void process() {
        if (dataBase != null) {
            dataBase.application();  // 문제 발생: AOP 미적용 가능성
        }
    }
}
```

---

## 💡 Solution

### 구조적 개선 포인트

- **Self-Bean 주입**
  - `@Autowired`를 사용하여 자기 자신을 주입 받아 내부 메서드를 호출하도록 수정.
  - 이를 통해 AOP 프록시가 정상적으로 적용되어 트랜잭션 및 비동기 기능이 동작.

---

## ✅ To-Be 구조

최종적으로 개선된 코드 구조는 다음과 같습니다:

```java
@BXMApplication
public class Test {

    @Autowired
    private Test self;  // Self-Bean 주입
    private DataBase dataBase;

    public void process() {
        if (dataBase != null) {
            self.callDatabase();  // 프록시를 통해 호출
        }
    }

    @Transactional
    public void callDatabase() {
        dataBase.application();  // AOP 적용됨
    }
}
```
---

## Self-Bean 주입을 통한 AOP 문제 해결

Self-Bean 주입을 통해 내부 메서드 호출을 프록시를 통해 수행하여 AOP 기능이 정상적으로 적용됨.  
`@Transactional` 어노테이션이 있는 메서드가 프록시를 거쳐 호출되므로 트랜잭션 관리가 정상적으로 작동합니다.  

---

## 📘 Lessons Learned

### 1. Self-Bean 주입을 통해 프록시 문제 해결
`@Autowired`로 자기 자신을 주입하여 내부 메서드를 호출하는 방식으로,  
프록시가 정상적으로 적용되어 AOP 기능이 제대로 동작함.  
이 방법을 통해 AOP 기능이 정상적으로 작동하고, 트랜잭션 관리 및 비동기 처리 기능이 기대한 대로 이루어졌음.

### 2. AOP와 프록시의 중요성
AOP 기능이 제대로 동작하려면 반드시 프록시 객체를 거쳐야 하며,  
내부에서 직접 호출할 경우 프록시가 무시되어 AOP가 적용되지 않음.  
따라서 AOP를 사용한 트랜잭션이나 비동기 처리를 위해서는 반드시 프록시를 경유하도록 구조를 설계해야 함.

### 3. 버전업 시 원인과 결과 발생 가능성
버전 업그레이드가 이루어지면 기존의 동작 방식과 달리 예상치 못한 문제가 발생할 수 있음.  
이 경우, A라는 원인(예: Spring AOP의 프록시 호출 방식 변화)으로  
B라는 결과(예: AOP 기능 미적용)가 발생할 수 있음.  
이러한 문제는 새로운 버전에서 바이트코드 최적화나 프록시 호출 방식이 변경되면서 발생하며,  
기존에 잘 작동하던 기능이 갑자기 동작하지 않을 수 있음.  

따라서 버전업 시 기존 동작 방식이 어떻게 변경되었는지 충분히 검토하고, 이에 맞게 구조를 조정하는 것이 중요함.  

---
