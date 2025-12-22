# Release-Please 설정 가이드 (Gradle 프로젝트)

이 문서는 java-test 프로젝트에서 release-please를 설정하면서 발생한 문제들과 해결 방법을 정리한 것입니다.
**실제 프로젝트(GroManage)에 적용 시 이 가이드를 참고하세요.**

---

## 📋 목차

1. [최종 작업 결과](#최종-작업-결과)
2. [발생했던 문제들과 해결 방법](#발생했던-문제들과-해결-방법)
3. [GroManage 프로젝트 적용 가이드](#gromanage-프로젝트-적용-가이드)
4. [필수 파일 구성](#필수-파일-구성)
5. [Workflow 설명](#workflow-설명)
6. [테스트 시나리오](#테스트-시나리오)
7. [트러블슈팅](#트러블슈팅)

---

## 최종 작업 결과

### ✅ 성공적으로 구축된 파일들

```
java-test/
├── .github/workflows/
│   ├── release-please.yml          # Release 자동화 workflow
│   ├── pr-build.yml                # PR 빌드/테스트
│   ├── pr-checks.yml               # PR 검증
│   ├── docker-publish.yml          # Docker 이미지 배포
│   └── labels.yml                  # 라벨 동기화
├── release-please-config.json      # Release-please 설정
├── .release-please-manifest.json   # 현재 버전 추적
├── build.gradle                    # version = '1.1.0'
├── CHANGELOG.md                    # 자동 생성된 변경 이력
├── Dockerfile                      # Docker 이미지 빌드용
└── TEST_SCENARIO.md                # 테스트 가이드
```

### 🎯 달성한 목표

- ✅ Conventional Commits 기반 자동 버전 관리
- ✅ CHANGELOG 자동 생성 (이모지 섹션 포함)
- ✅ GitHub Release 자동 생성
- ✅ Docker 이미지 자동 빌드 및 GHCR 배포
- ✅ develop → main PR flow 구축

---

## 발생했던 문제들과 해결 방법

### 🔴 문제 1: Deprecated Action 사용

**에러:**
```
google-github-actions/release-please-action is deprecated
```

**원인:**
- 기존에 deprecated된 `google-github-actions/release-please-action@v4` 사용

**해결:**
```yaml
# ❌ 잘못된 방법
uses: google-github-actions/release-please-action@v4

# ✅ 올바른 방법
uses: googleapis/release-please-action@v4
```

---

### 🔴 문제 2: Package-name과 Changelog-types 파라미터 오류

**에러:**
```
Unexpected input(s) 'package-name', 'changelog-types'
```

**원인:**
- v4에서는 이 파라미터들을 직접 지원하지 않음
- config 파일을 통해 설정해야 함

**해결:**
```yaml
# ❌ 잘못된 방법 (workflow에 직접 작성)
with:
  release-type: java
  package-name: gromanage
  changelog-types: |
    [{"type":"feat","section":"Features"}]

# ✅ 올바른 방법 (config 파일 사용)
with:
  config-file: release-please-config.json
  manifest-file: .release-please-manifest.json
```

---

### 🔴 문제 3: Tag 형식 불일치

**에러:**
```
looking for tagName: gromanage-v1.0.0
Expected tag: v1.0.0
```

**원인:**
- config에 `package-name: gromanage` 설정 시 태그에 패키지명 포함
- 실제 태그는 `v1.0.0` 형식

**해결:**
```json
// ❌ 잘못된 방법
{
  "packages": {
    ".": {
      "release-type": "simple",
      "package-name": "gromanage"  // 제거 필요!
    }
  }
}

// ✅ 올바른 방법 (단일 패키지 프로젝트)
{
  "packages": {
    ".": {
      "release-type": "simple"
      // package-name 없음
    }
  }
}
```

---

### 🔴 문제 4: 오래된 release-please 브랜치

**에러:**
```
release-please failed: Not Found
```

**원인:**
- 이전 release-please action이 생성한 `release-please--branches--main` 브랜치가 남아있음

**해결:**
```bash
# 오래된 브랜치 삭제
git push origin --delete release-please--branches--main
```

---

### 🔴 문제 5: Java Release Type과 Gradle 불일치 (핵심!)

**에러:**
```
Empty change set provided. No changes need to be made.
Repository needs a snapshot bump.
```

**원인:**
- **Java release type은 Maven 프로젝트용** (pom.xml 자동 업데이트)
- Gradle 프로젝트에서는 `extra-files` 설정이 없으면 어떤 파일도 업데이트하지 못함
- Java release type이 "snapshot bump" 생성을 시도하지만 실패

**해결:**
```json
// ❌ 잘못된 방법 (Gradle 프로젝트에서 Java 사용)
{
  "packages": {
    ".": {
      "release-type": "java"  // Maven 전용!
    }
  }
}

// ✅ 올바른 방법 (Gradle 프로젝트에서 Simple 사용)
{
  "packages": {
    ".": {
      "release-type": "simple"  // CHANGELOG와 manifest만 관리
    }
  }
}
```

**Release Type 비교:**

| Release Type | 용도 | 자동 업데이트 파일 | Gradle 프로젝트 |
|--------------|------|-------------------|----------------|
| `java` | Maven 프로젝트 | pom.xml | ❌ extra-files 필요 |
| `maven` | Maven 프로젝트 | pom.xml (재귀적) | ❌ extra-files 필요 |
| `simple` | 범용 | CHANGELOG, manifest | ✅ 추천! |

---

### 🔴 문제 6: build.gradle 버전 불일치

**상황:**
- `.release-please-manifest.json`: `"1.0.0"`
- `build.gradle`: `version = '0.1.0'`
- GitHub Release: `v1.0.0`

**해결:**
```gradle
// build.gradle에서 버전 동기화
group = 'com.example'
version = '1.0.0'  // manifest와 일치시킴
```

**참고:** Simple release type 사용 시 build.gradle 버전은 수동 관리 필요

---

## GroManage 프로젝트 적용 가이드

### 📌 현재 GroManage 상태 (확인 필요)

```bash
# build.gradle 확인
group = 'com.grolabs'
version = '0.0.1-SNAPSHOT'  # ← 초기 버전 설정 필요

# 브랜치 구조
- main: 내용 거의 없음
- develop: 실제 개발 코드 대부분 포함
```

### 🚀 적용 단계

#### 1단계: 초기 Release 생성 (main 브랜치)

**목표:** develop의 내용을 main에 병합하고 첫 릴리즈(v1.0.0) 생성

```bash
# 1. main 브랜치에서 시작
git checkout main
git pull origin main

# 2. develop의 변경사항을 main으로 병합
git merge develop
# 또는 GitHub에서 develop → main PR 생성 후 병합

# 3. build.gradle 버전 확인 및 수정
# version = '0.0.1-SNAPSHOT' → version = '1.0.0'로 변경

# 4. 수동으로 첫 릴리즈 태그 생성
git tag v1.0.0
git push origin v1.0.0

# 5. GitHub에서 Release 수동 생성 (v1.0.0)
# - 태그: v1.0.0
# - 제목: "v1.0.0"
# - 내용: 초기 릴리즈 설명
```

#### 2단계: Release-Please 파일 추가

```bash
# 1. release-please-config.json 생성
cat > release-please-config.json << 'EOF'
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "release-type": "simple",
      "changelog-sections": [
        {"type": "feat", "section": "✨ Features", "hidden": false},
        {"type": "fix", "section": "🐛 Bug Fixes", "hidden": false},
        {"type": "perf", "section": "⚡ Performance Improvements", "hidden": false},
        {"type": "refactor", "section": "♻️ Code Refactoring", "hidden": false},
        {"type": "docs", "section": "📚 Documentation", "hidden": false},
        {"type": "build", "section": "🏗️ Build System", "hidden": false},
        {"type": "ci", "section": "🔧 CI/CD", "hidden": false},
        {"type": "chore", "section": "🧹 Chores", "hidden": false},
        {"type": "revert", "section": "⏪ Reverts", "hidden": false}
      ]
    }
  }
}
EOF

# 2. .release-please-manifest.json 생성 (현재 버전 명시)
cat > .release-please-manifest.json << 'EOF'
{
  ".": "1.0.0"
}
EOF

# 3. CHANGELOG.md 생성 (초기 버전)
cat > CHANGELOG.md << 'EOF'
# Changelog

## 1.0.0 (YYYY-MM-DD)

### Features

* 초기 릴리즈
EOF
```

#### 3단계: GitHub Workflows 추가

**필수 Workflow 파일들을 `.github/workflows/` 에 생성:**

1. **release-please.yml** (핵심!)
2. **pr-build.yml** (PR 빌드/테스트)
3. **pr-checks.yml** (PR 검증)
4. **docker-publish.yml** (선택사항 - Docker 이미지 배포)
5. **labels.yml** (선택사항 - 라벨 동기화)

**각 파일은 java-test 프로젝트에서 복사 가능**

#### 4단계: Dockerfile 추가 (선택사항)

GroManage는 Spring Boot 프로젝트이므로 Dockerfile 필요:

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
EXPOSE 8080
```

#### 5단계: 커밋 및 푸시

```bash
git add release-please-config.json .release-please-manifest.json CHANGELOG.md
git add .github/workflows/
git add Dockerfile  # Dockerfile 추가한 경우

git commit -m "chore: setup release-please automation

- Add release-please config for automatic versioning
- Add GitHub workflows for CI/CD
- Add Dockerfile for container deployment
- Initial version: 1.0.0"

git push origin main
```

#### 6단계: 테스트

**develop에서 기능 개발 후 테스트:**

```bash
# 1. develop 브랜치에서 기능 개발
git checkout develop
echo "test" >> README.md
git commit -m "feat: add new feature"
git push origin develop

# 2. develop → main PR 생성 및 병합

# 3. main에 병합되면 release-please workflow 자동 실행
# - v1.1.0 Release PR 자동 생성
# - CHANGELOG 업데이트
# - manifest 업데이트 (1.0.0 → 1.1.0)

# 4. Release PR 머지
# - GitHub Release 자동 생성
# - Docker 이미지 자동 빌드/배포 (Dockerfile 있는 경우)
```

---

## 필수 파일 구성

### 1. release-please-config.json

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "release-type": "simple",
      "changelog-sections": [
        {"type": "feat", "section": "✨ Features", "hidden": false},
        {"type": "fix", "section": "🐛 Bug Fixes", "hidden": false},
        {"type": "perf", "section": "⚡ Performance Improvements", "hidden": false},
        {"type": "refactor", "section": "♻️ Code Refactoring", "hidden": false},
        {"type": "docs", "section": "📚 Documentation", "hidden": false},
        {"type": "build", "section": "🏗️ Build System", "hidden": false},
        {"type": "ci", "section": "🔧 CI/CD", "hidden": false},
        {"type": "chore", "section": "🧹 Chores", "hidden": false},
        {"type": "revert", "section": "⏪ Reverts", "hidden": false}
      ]
    }
  }
}
```

**설정 설명:**
- `release-type: "simple"`: Gradle 프로젝트에 적합 (CHANGELOG, manifest만 관리)
- `changelog-sections`: CHANGELOG에 표시될 커밋 타입 정의
- **중요:** `package-name` 필드는 제거 (단일 패키지 프로젝트)

### 2. .release-please-manifest.json

```json
{
  ".": "1.0.0"
}
```

**설명:**
- 현재 릴리즈된 버전 추적
- Release PR 생성 시 자동 업데이트됨
- **초기값을 현재 버전과 일치시켜야 함!**

### 3. .github/workflows/release-please.yml

```yaml
name: Release Please

on:
  push:
    branches:
      - main

env:
  JAVA_VERSION: '21'
  JAVA_DISTRIBUTION: 'temurin'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: write
  pull-requests: write
  packages: write

jobs:
  release-please:
    name: Create Release
    runs-on: ubuntu-latest
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
      version: ${{ steps.release.outputs.major }}.${{ steps.release.outputs.minor }}.${{ steps.release.outputs.patch }}

    steps:
      - name: Run Release Please
        id: release
        uses: googleapis/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json

      - name: Output Release Info
        if: steps.release.outputs.release_created
        run: |
          echo "✅ Release created!"
          echo "Tag: ${{ steps.release.outputs.tag_name }}"
          echo "Version: ${{ steps.release.outputs.major }}.${{ steps.release.outputs.minor }}.${{ steps.release.outputs.patch }}"

  build-and-push-docker:
    name: Build and Push Docker Image
    needs: release-please
    if: needs.release-please.outputs.release_created == 'true'
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}

      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Make gradlew executable
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew build -x test --no-daemon

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}},value=${{ needs.release-please.outputs.tag_name }}
            type=semver,pattern={{major}}.{{minor}},value=${{ needs.release-please.outputs.tag_name }}
            type=semver,pattern={{major}},value=${{ needs.release-please.outputs.tag_name }}
            type=raw,value=latest

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64

      - name: Update release with Docker image info
        uses: actions/github-script@v7
        with:
          script: |
            const release = await github.rest.repos.getReleaseByTag({
              owner: context.repo.owner,
              repo: context.repo.repo,
              tag: '${{ needs.release-please.outputs.tag_name }}'
            });

            const imageUrl = `${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}`;
            const version = '${{ needs.release-please.outputs.tag_name }}';

            const additionalInfo = `

            ## 🐳 Docker 이미지

            이 릴리즈의 Docker 이미지는 GitHub Container Registry에서 다운로드할 수 있습니다:

            \`\`\`bash
            docker pull ${imageUrl}:${version}
            docker pull ${imageUrl}:latest
            \`\`\`

            ### 사용 방법

            \`\`\`bash
            # 이미지 다운로드
            docker pull ${imageUrl}:${version}

            # 컨테이너 실행
            docker run -p 8080:8080 ${imageUrl}:${version}
            \`\`\`

            ### 이미지 정보
            - Registry: GitHub Container Registry (ghcr.io)
            - Repository: ${imageUrl}
            - Tags: \`${version}\`, \`latest\`
            - Platform: linux/amd64
            `;

            await github.rest.repos.updateRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              release_id: release.data.id,
              body: release.data.body + additionalInfo
            });

      - name: Output image info
        run: |
          echo "✅ Docker 이미지가 성공적으로 빌드되고 배포되었습니다!"
          echo "Registry: ${{ env.REGISTRY }}"
          echo "Image: ${{ env.IMAGE_NAME }}"
          echo "Tag: ${{ needs.release-please.outputs.tag_name }}"
```

---

## Workflow 설명

### 전체 Flow

```
1. Feature 개발 (develop 브랜치)
   ↓
2. develop → main PR 생성 (Conventional Commit 메시지)
   ↓
3. PR 병합 → main 브랜치에 push
   ↓
4. release-please workflow 자동 실행
   - 커밋 분석 (v1.0.0 이후)
   - 버전 계산 (feat = minor↑, fix = patch↑)
   - Release PR 자동 생성
   ↓
5. Release PR 검토 및 머지
   ↓
6. GitHub Release 자동 생성
   ↓
7. Docker 이미지 빌드 및 GHCR 배포
```

### Conventional Commits 규칙

| 타입 | 버전 영향 | 설명 |
|------|----------|------|
| `feat:` | minor ↑ | 새로운 기능 추가 |
| `fix:` | patch ↑ | 버그 수정 |
| `perf:` | patch ↑ | 성능 개선 |
| `refactor:` | - | 코드 리팩토링 |
| `docs:` | - | 문서 수정 |
| `build:` | - | 빌드 시스템 변경 |
| `ci:` | - | CI 설정 변경 |
| `chore:` | - | 기타 변경사항 |
| `revert:` | - | 커밋 되돌리기 |

**BREAKING CHANGE:**
- 커밋 메시지에 `BREAKING CHANGE:` 포함 시 → major 버전 증가

---

## 테스트 시나리오

### 시나리오 1: Feature 추가 (Minor 버전 증가)

```bash
# 1. develop에서 기능 개발
git checkout develop
# ... 코드 작업 ...
git commit -m "feat: add user authentication"
git push origin develop

# 2. develop → main PR 생성 및 병합
# PR 제목: "feat: add user authentication"

# 3. main 병합 후 자동 실행
# - Release PR 생성: "chore(main): release 1.1.0"
# - CHANGELOG 업데이트

# 4. Release PR 머지
# - v1.1.0 Release 생성
# - Docker 이미지 배포
```

### 시나리오 2: 버그 수정 (Patch 버전 증가)

```bash
# 1. develop에서 버그 수정
git checkout develop
# ... 버그 수정 ...
git commit -m "fix: resolve login error"
git push origin develop

# 2. develop → main PR 생성 및 병합

# 3. Release PR 생성: "chore(main): release 1.1.1"
```

### 시나리오 3: Breaking Change (Major 버전 증가)

```bash
git commit -m "feat!: redesign API structure

BREAKING CHANGE: API endpoints have been completely redesigned"

# Release PR: "chore(main): release 2.0.0"
```

---

## 트러블슈팅

### ❌ "Empty change set" 에러

**원인:**
- Java release type을 Gradle 프로젝트에서 사용
- 버전 불일치 (manifest vs build.gradle)

**해결:**
1. `release-type`을 `simple`로 변경
2. build.gradle 버전을 manifest와 동기화

### ❌ "Not Found" 에러

**원인:**
- 오래된 release-please 브랜치 존재

**해결:**
```bash
git push origin --delete release-please--branches--main
```

### ❌ Tag 형식 불일치

**원인:**
- config에 `package-name` 설정

**해결:**
- `package-name` 필드 제거 (단일 패키지 프로젝트)

### ❌ Workflow가 실행되지 않음

**확인 사항:**
1. `.github/workflows/release-please.yml` 파일 존재 확인
2. main 브랜치에 push 되었는지 확인
3. GitHub Actions 권한 설정 확인
   - Settings → Actions → General → Workflow permissions
   - "Read and write permissions" 체크

---

## 체크리스트

### GroManage 적용 전 체크리스트

- [ ] main 브랜치에 develop 내용 병합 완료
- [ ] build.gradle 버전을 1.0.0으로 설정
- [ ] v1.0.0 태그 생성
- [ ] GitHub Release v1.0.0 생성
- [ ] release-please-config.json 생성
- [ ] .release-please-manifest.json 생성 (1.0.0)
- [ ] CHANGELOG.md 초기 버전 생성
- [ ] .github/workflows/release-please.yml 생성
- [ ] Dockerfile 생성 (Docker 이미지 필요 시)
- [ ] 모든 파일 커밋 및 push
- [ ] Workflow 테스트 (develop에서 feat 커밋 → main PR)

### 적용 후 검증 체크리스트

- [ ] develop → main PR 시 release-please workflow 실행됨
- [ ] Release PR 자동 생성됨
- [ ] CHANGELOG 업데이트 확인 (이모지 섹션 포함)
- [ ] Release PR 머지 시 GitHub Release 생성됨
- [ ] Docker 이미지 GHCR에 배포됨 (Dockerfile 있는 경우)
- [ ] Release notes에 Docker 사용 방법 추가됨

---

## 참고 자료

- [googleapis/release-please-action](https://github.com/googleapis/release-please-action)
- [release-please Java 문서](https://github.com/googleapis/release-please/blob/main/docs/java.md)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [java-test 프로젝트 TEST_SCENARIO.md](./TEST_SCENARIO.md)

---

## 작성 정보

- 작성일: 2025-12-22
- 프로젝트: java-test (테스트 프로젝트)
- 목적: GroManage 실제 프로젝트 적용을 위한 가이드
- 작성자: Claude Code

---

## 다음 단계 (GroManage 적용)

1. **새 채팅 세션 시작**
   - Context: `/home/vinyl/project/GroManage`

2. **이 가이드 참고하여 단계별 적용**
   - 초기 릴리즈 설정
   - 파일 생성
   - Workflow 테스트

3. **문제 발생 시**
   - 이 문서의 트러블슈팅 섹션 참고
   - java-test 프로젝트의 설정 파일 참고
