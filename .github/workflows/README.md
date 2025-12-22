# Java (Spring Boot, Gradle) Workflows

이 디렉토리는 **Java (Spring Boot, Gradle)** 프로젝트를 위한 GitHub Actions Workflow 템플릿을 제공합니다.

---

## 📋 포함된 Workflow

| Workflow | 파일명 | 트리거 | 설명 |
|----------|--------|--------|------|
| **PR Build & Test** | `pr-build.yml` | PR 생성/업데이트 | PR 빌드, 테스트, Docker 검증 |
| **Docker Publish** | `docker-publish.yml` | Release 생성 | Docker 이미지 빌드 & GHCR 배포 |
| **Deploy Example** | `deploy-example.yml` | - | Self-hosted runner 배포 예시 (참고용) |

---

## 🚀 빠른 시작

### 1. 필수 요구사항

#### 프로젝트 설정
- **Gradle Wrapper** 필수 (`gradlew`, `gradle/wrapper/`)
- **`build.gradle` 또는 `build.gradle.kts`** 필수
- Gradle Toolchain 설정 권장 (버전 명시)

#### build.gradle 예시 (Gradle Toolchain 사용)
```gradle
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

또는 build.gradle.kts:
```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}
```

### 2. Workflow 복사

```bash
# 프로젝트로 이동
cd /path/to/your-project

# Java workflow 복사
cp /path/to/template/.github/workflows/java/* .github/workflows/
cp /path/to/template/.github/workflows/_common/* .github/workflows/
```

### 3. Java 버전 설정

각 workflow 파일 상단의 `env` 섹션에서 Java 버전을 설정합니다.

```yaml
env:
  JAVA_VERSION: '21'  # 프로젝트의 Java 버전으로 변경
  JAVA_DISTRIBUTION: 'temurin'
```

**중요**: 이 버전은 `build.gradle`의 Toolchain 설정과 일치해야 합니다!

### 4. Docker 설정 (선택사항)

Docker 이미지 빌드를 사용하는 경우:
- 프로젝트 루트에 `Dockerfile` 생성
- `docker-publish.yml`의 `IMAGE_NAME`과 `REGISTRY` 확인

---

## 📝 Workflow 상세 설명

### PR Build & Test (`pr-build.yml`)

**트리거**:
- `develop` 또는 `main` 브랜치로의 PR 생성/업데이트

**실행 내용**:
1. Java 설정 (Toolchain 버전 사용)
2. Gradle 캐시 복원
3. `./gradlew build -x test` (빌드만)
4. `./gradlew test` (테스트 실행)
5. Dockerfile이 있으면 Docker 이미지 빌드 검증
6. 실패 시 빌드 리포트 업로드

**커스터마이징**:
```yaml
# 특정 브랜치만 트리거
on:
  pull_request:
    branches:
      - develop
      - main
      - feature/*

# Java 버전 변경
env:
  JAVA_VERSION: '17'  # Java 17 사용

# 추가 테스트 옵션
- name: Run tests
  run: ./gradlew test --no-daemon --parallel
```

---

### Docker Publish (`docker-publish.yml`)

**트리거**:
- GitHub Release 생성 시
- 수동 실행 (`workflow_dispatch`)

**실행 내용**:
1. Java 설정
2. Gradle 빌드 (`./gradlew build -x test`)
3. Docker 이미지 빌드
4. GitHub Container Registry(GHCR)에 푸시
5. Release 노트에 Docker 이미지 정보 추가

**이미지 태그**:
- `latest` (기본 브랜치)
- `vX.Y.Z` (Release 태그)
- `vX.Y` (Major.Minor)
- `vX` (Major)

**커스터마이징**:
```yaml
# 다른 레지스트리 사용 (예: Docker Hub)
env:
  REGISTRY: docker.io
  IMAGE_NAME: your-dockerhub-username/your-app

# Multi-platform 빌드
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
```

---

## 🔧 버전 관리 전략

### 기본 원칙

1. **개발 중에는 버전 변경하지 않음**
   - `build.gradle`의 `version` 필드를 직접 수정하지 않음
   - Git tag가 버전의 단일 기준(SSOT)

2. **릴리즈 시점에 Git tag로 버전 관리**
   - Release-Please가 Conventional Commits 기반으로 버전 자동 결정
   - GitHub Release 생성 시 Git tag 자동 생성

### Conventional Commits 예시

```bash
# Minor 버전 증가 (1.0.0 → 1.1.0)
git commit -m "feat: 새로운 API 엔드포인트 추가"

# Patch 버전 증가 (1.0.0 → 1.0.1)
git commit -m "fix: NPE 버그 수정"

# Major 버전 증가 (1.0.0 → 2.0.0)
git commit -m "feat!: API 응답 형식 변경

BREAKING CHANGE: /api/users 엔드포인트의 응답 형식이 변경되었습니다."
```

---

## 🐳 Docker 이미지 사용법

Release가 생성되면 자동으로 GHCR에 Docker 이미지가 배포됩니다.

```bash
# 이미지 다운로드
docker pull ghcr.io/your-org/your-repo:v1.2.3

# 컨테이너 실행
docker run -p 8080:8080 ghcr.io/your-org/your-repo:v1.2.3

# 환경 변수 설정
docker run -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DATABASE_URL=jdbc:postgresql://localhost:5432/mydb \
  ghcr.io/your-org/your-repo:v1.2.3
```

---

## 🔍 트러블슈팅

### Q1. Java 버전 불일치 오류

**증상**:
```
Execution failed for task ':compileJava'.
> Could not target platform: 'Java SE 21' using tool chain: 'JDK 17 (17.0.x)'.
```

**해결**:
- workflow의 `JAVA_VERSION`과 `build.gradle`의 Toolchain 버전을 일치시킵니다.

### Q2. Gradle 캐시 미작동

**증상**:
- 의존성 다운로드가 매번 실행됨

**해결**:
```yaml
- name: Cache Gradle packages
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: |
      ${{ runner.os }}-gradle-
```

### Q3. Docker 이미지 빌드 실패

**증상**:
```
ERROR: failed to solve: Dockerfile: no such file or directory
```

**해결**:
- 프로젝트 루트에 `Dockerfile`이 있는지 확인
- 또는 `pr-build.yml`에서 Docker 검증 단계 제거

---

## 📚 추가 리소스

- [Gradle Toolchain 공식 문서](https://docs.gradle.org/current/userguide/toolchains.html)
- [GitHub Actions - setup-java](https://github.com/actions/setup-java)
- [GitHub Container Registry 가이드](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

---

## 💬 문의

궁금한 점이 있다면 [Issue](../../issues)를 생성해주세요!
