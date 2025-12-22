# Java Test Project - 테스트 시나리오

이 프로젝트는 Java (Spring Boot, Gradle) workflow를 테스트하기 위한 최소 프로젝트입니다.

---

## 📋 프로젝트 구조

```
java-test/
├── .github/
│   ├── workflows/
│   │   ├── pr-build.yml          # PR 빌드 & 테스트
│   │   ├── docker-publish.yml    # Docker 이미지 배포
│   │   ├── deploy-example.yml    # 참고용 배포 예시
│   │   ├── labels.yml            # 라벨 동기화
│   │   ├── pr-checks.yml         # PR 검증
│   │   └── release-please.yml    # 릴리즈 자동화
│   ├── ISSUE_TEMPLATE/           # Issue 템플릿
│   ├── labels.yml
│   └── pull_request_template.md
├── src/
│   ├── main/java/com/example/
│   │   └── Application.java      # Spring Boot 메인 클래스
│   └── test/java/com/example/
│       └── ApplicationTests.java # 테스트
├── gradle/
│   └── wrapper/
├── build.gradle                  # 빌드 설정
└── settings.gradle
```

---

## 🚀 로컬 테스트

### 1. Gradle 빌드 테스트

```bash
# Gradle wrapper 다운로드 (처음 한 번만)
gradle wrapper

# 빌드 (테스트 제외)
./gradlew build -x test

# 테스트 실행
./gradlew test

# 애플리케이션 실행
./gradlew bootRun
# 브라우저에서 http://localhost:8080 접속
```

---

## 🧪 GitHub Actions 테스트 시나리오

### 시나리오 1: PR Build Workflow 테스트

**목적**: PR 생성 시 자동 빌드 & 테스트가 실행되는지 확인

**단계**:

```bash
# 1. GitHub에 새 레포지토리 생성
# - 레포 이름: java-test-workflow (또는 원하는 이름)

# 2. 로컬에서 Git 초기화
cd java-test
git init
git add .
git commit -m "feat: initial Spring Boot project"
git branch -M main

# 3. 원격 레포지토리 연결 및 푸시
git remote add origin https://github.com/YOUR_USERNAME/java-test-workflow.git
git push -u origin main

# 4. develop 브랜치 생성
git checkout -b develop
git push origin develop

# 5. 기능 브랜치 생성 및 변경
git checkout -b feat/test-pr-workflow

# 6. 코드 수정 (예: Application.java)
# Application.java의 hello() 메서드 수정
echo '
    @GetMapping("/test")
    public String test() {
        return "Test endpoint";
    }
' >> src/main/java/com/example/Application.java

git add .
git commit -m "feat: 테스트 엔드포인트 추가"
git push origin feat/test-pr-workflow

# 7. GitHub에서 PR 생성
# - Base: develop
# - Compare: feat/test-pr-workflow
# - PR 제목: "feat: 테스트 엔드포인트 추가"
# - PR 본문: "Fixes #1" (Issue를 먼저 생성해야 함)
```

**확인 사항**:
- [ ] `pr-build.yml` workflow가 자동 실행됨
- [ ] Java 21 설정 성공
- [ ] Gradle 빌드 성공
- [ ] 테스트 통과
- [ ] `pr-checks.yml`에서 PR 제목 검증 성공
- [ ] `pr-checks.yml`에서 Issue 연결 검증 성공 (Fixes #N 있어야 함)

---

### 시나리오 2: Conventional Commits 검증 테스트

**목적**: PR 제목이 Conventional Commits 형식을 따르는지 검증

**단계**:

```bash
# 잘못된 PR 제목으로 PR 생성
# ❌ "Update code"
# ❌ "Feat: Add feature" (대문자 시작)
# ✅ "feat: 기능 추가" (올바른 형식)
```

**확인 사항**:
- [ ] 잘못된 형식의 PR 제목은 검증 실패
- [ ] 올바른 형식의 PR 제목은 검증 통과

---

### 시나리오 3: Issue 연결 검증 테스트

**목적**: PR이 Issue와 연결되어 있는지 확인

**단계**:

```bash
# 1. GitHub에서 Issue 생성
# - 제목: "테스트용 Issue"
# - 번호 확인 (예: #1)

# 2. PR 본문에 Issue 연결
# Fixes #1
# 또는
# Closes #1
# 또는
# Resolves #1
```

**확인 사항**:
- [ ] Issue 연결이 없으면 검증 실패
- [ ] Issue 연결이 있으면 검증 통과

---

### 시나리오 4: Release-Please 테스트

**목적**: Conventional Commits 기반 자동 릴리즈 생성

**사전 설정**:
- `googleapis/release-please-action@v4` 사용 (최신 유지보수 버전)
- `release-please-config.json`: changelog 커스터마이징 설정
- `.release-please-manifest.json`: 현재 버전 추적

**단계**:

```bash
# 1. develop 브랜치에 여러 커밋 추가
git checkout develop
git pull origin develop

git commit --allow-empty -m "feat: 새로운 기능 1"
git commit --allow-empty -m "feat: 새로운 기능 2"
git commit --allow-empty -m "fix: 버그 수정"
git push origin develop

# 2. develop → main PR 생성
# GitHub에서:
# - Base: main
# - Compare: develop
# - PR 제목: "chore: release 준비"
# - Merge

# 3. main 브랜치에 push되면 Release-Please workflow 자동 실행
# - workflow가 자동으로 Release PR 생성
# - PR 제목: "chore(main): release 1.x.x" (버전은 자동 결정)
# - CHANGELOG 자동 생성 확인

# 4. Release PR 머지
# - 머지 시 자동으로 GitHub Release 생성
# - Docker 이미지 빌드 및 GHCR에 푸시
# - Release notes에 Docker 이미지 정보 추가

# 5. GitHub Release 자동 생성 확인
# - Tag: v1.x.x
# - Release notes에 CHANGELOG 포함
# - Docker 이미지 다운로드 명령어 포함
```

**확인 사항**:
- [ ] Release-Please PR 자동 생성 (googleapis/release-please-action@v4 사용)
- [ ] CHANGELOG에 feat, fix 커밋 포함 (이모지 섹션 포함)
- [ ] 버전 번호 자동 결정 (feat = minor 증가, fix = patch 증가)
- [ ] GitHub Release 자동 생성
- [ ] Git tag (v1.x.x) 생성
- [ ] Docker 이미지 GHCR 배포 성공

---

### 시나리오 5: Docker 이미지 배포 테스트

**참고**: Docker 이미지 빌드 및 배포는 **시나리오 4에 통합**되었습니다.

**통합된 기능**:
- Release 생성 시 `release-please.yml` workflow에서 자동으로 Docker 이미지 빌드
- GHCR(GitHub Container Registry)에 자동 푸시
- Release notes에 Docker 이미지 사용 방법 자동 추가

**Dockerfile 확인**:
프로젝트에 이미 `Dockerfile`이 포함되어 있습니다:
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
EXPOSE 8080
```

**확인 사항** (시나리오 4와 함께 확인):
- [ ] Release 생성 시 Docker 이미지 빌드 성공
- [ ] GHCR에 이미지 푸시 성공
- [ ] 이미지 태그 생성:
  - `ghcr.io/YOUR_USERNAME/java-workflow-test:v1.x.x`
  - `ghcr.io/YOUR_USERNAME/java-workflow-test:latest`
- [ ] Release notes에 Docker 이미지 정보 자동 추가

---

## 🔍 트러블슈팅

### 문제 1: Gradle wrapper 실행 권한 오류

**증상**:
```
Permission denied: ./gradlew
```

**해결**:
```bash
chmod +x gradlew
git add gradlew
git commit -m "fix: gradlew 실행 권한 추가"
```

---

### 문제 2: Java 버전 불일치

**증상**:
```
Execution failed for task ':compileJava'.
> Could not target platform: 'Java SE 21'
```

**해결**:
- `build.gradle`의 `toolchain` 설정 확인
- `.github/workflows/java/pr-build.yml`의 `JAVA_VERSION` 확인
- 두 값이 일치해야 함 (현재: 21)

---

### 문제 3: PR 검증 실패 - Issue 연결 없음

**증상**:
```
❌ PR 본문에 연결된 Issue가 없습니다.
```

**해결**:
- PR 본문에 `Fixes #N` 추가
- 또는 `Closes #N`, `Resolves #N`

---

## ✅ 최종 체크리스트

전체 테스트 완료 확인:

- [ ] 로컬 빌드 성공
- [ ] 로컬 테스트 통과
- [ ] PR Build workflow 성공
- [ ] PR Checks workflow 성공 (Conventional Commits)
- [ ] PR Checks workflow 성공 (Issue 연결)
- [ ] Release-Please PR 생성 확인
- [ ] GitHub Release 생성 확인
- [ ] Docker 이미지 배포 성공 (선택사항)
- [ ] Labels 동기화 확인

---

## 📚 참고 자료

- [Java Workflow 상세 가이드](../.github/workflows/java/README.md)
- [CONTRIBUTING.md](../.github/CONTRIBUTING.md)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Release-Please](https://github.com/googleapis/release-please)

---

## 💡 팁

1. **Issue 먼저 생성**: PR 전에 Issue를 생성하면 PR Checks가 통과됩니다
2. **Conventional Commits 사용**: `feat:`, `fix:` 등으로 시작하는 커밋 메시지 사용
3. **develop 브랜치 활용**: feature 브랜치 → develop → main 흐름 유지
4. **로컬 테스트 먼저**: GitHub Actions 실행 전 로컬에서 먼저 테스트

즐거운 테스트 되세요! 🚀
