# Java Test Project

Java (Spring Boot, Gradle) workflow 테스트를 위한 최소 프로젝트입니다.

## 빠른 시작

```bash
# Gradle wrapper 생성
gradle wrapper

# 빌드
./gradlew build

# 테스트
./gradlew test

# 실행
./gradlew bootRun
```

## 테스트 시나리오

자세한 테스트 시나리오는 [TEST_SCENARIO.md](TEST_SCENARIO.md)를 참고하세요.

## 엔드포인트

- `GET /` - Hello World
- `GET /health` - Health Check
