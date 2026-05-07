---
name: repo-safety-check
description: "GitHub 저장소 클론 전 보안 검증. 위험 스크립트, 의심 코드 패턴 탐지. '레포 안전해?', 'safe to clone?', '저장소 확인' 등 요청 시 사용"
triggers:
  - 레포 안전
  - 레포 검증
  - repo safety
  - safe to clone
  - 저장소 확인
  - check repo
  - 클론해도 될까
  - 이 레포 괜찮아
---

# Repo Safety Check

GitHub 저장소를 클론하기 전 보안 위험을 사전에 탐지합니다.

## 사용법

```
/repo-safety-check https://github.com/owner/repo
/repo-safety-check owner/repo
```

## 워크플로우

### Phase 1: 환경 확인

gh CLI 설치 여부를 확인합니다.

```bash
command -v gh
```

**gh CLI 미설치 시:**

설치 가이드를 제공하고 curl 대안을 사용합니다.

| OS | 설치 명령 |
|----|----------|
| macOS | `brew install gh` |
| Ubuntu/Debian | `sudo apt install gh` |
| Windows | `winget install GitHub.cli` |

curl 대안 (public repo만, rate limit 60회/시간):
```bash
curl -s "https://api.github.com/repos/OWNER/REPO"
curl -s "https://raw.githubusercontent.com/OWNER/REPO/main/FILE"
```

### Phase 2: URL 파싱

입력에서 `owner/repo` 추출:

```bash
# 전체 URL에서 추출
echo "https://github.com/owner/repo.git" | sed -E 's|.*github\.com[/:]([^/]+)/([^/.]+).*|\1/\2|'

# 이미 owner/repo 형식이면 그대로 사용
```

### Phase 3: 메타데이터 수집

저장소 신뢰도 지표를 수집합니다.

```bash
gh api repos/OWNER/REPO --jq '{
  stars: .stargazers_count,
  forks: .forks_count,
  updated: .updated_at,
  created: .created_at,
  owner_type: .owner.type,
  license: .license.spdx_id,
  open_issues: .open_issues_count,
  archived: .archived
}'
```

**신뢰도 평가 기준:**

| 지표 | 안전 | 주의 | 위험 |
|------|------|------|------|
| Stars | 100+ | 10-99 | <10 |
| Last Update | <30일 | 30-180일 | >180일 |
| Owner Type | Organization | User (verified) | User (new) |
| License | OSI 승인 | 기타 | 없음 |

### Phase 4: 파일 목록 스캔

재귀적으로 모든 파일 목록을 조회합니다.

```bash
gh api repos/OWNER/REPO/git/trees/HEAD?recursive=1 --jq '.tree[].path'
```

**위험 파일 필터링:**

| 위험도 | 파일 패턴 |
|--------|----------|
| HIGH | `*.sh`, `*.bat`, `*.ps1`, `*.cmd` |
| HIGH | `postinstall`, `preinstall` in scripts |
| MEDIUM | `Makefile`, `setup.py`, `install.py`, `configure` |
| LOW | `.github/workflows/*.yml` |

### Phase 5: 위험 패턴 분석

의심 파일의 내용을 확인하고 위험 패턴을 탐지합니다.

```bash
# 파일 내용 조회
gh api repos/OWNER/REPO/contents/FILE_PATH -H "Accept:application/vnd.github.raw"
```

**탐지할 코드 패턴:**

| 카테고리 | 패턴 | 위험도 | 설명 |
|----------|------|--------|------|
| 동적 실행 | `eval(` | HIGH | 동적 코드 실행 |
| 동적 실행 | `exec(` | HIGH | 외부 명령 실행 |
| 프로세스 | `subprocess` | MEDIUM | Python 프로세스 생성 |
| 프로세스 | `child_process` | MEDIUM | Node.js 프로세스 생성 |
| 프로세스 | `os.system` | HIGH | 시스템 명령 실행 |
| 원격 실행 | `curl \| sh` | CRITICAL | 원격 스크립트 실행 |
| 원격 실행 | `curl \| bash` | CRITICAL | 원격 스크립트 실행 |
| 원격 실행 | `wget -O - \|` | CRITICAL | 원격 스크립트 실행 |
| 난독화 | `atob(` | HIGH | Base64 디코딩 |
| 난독화 | `String.fromCharCode` | HIGH | 문자열 난독화 |
| 난독화 | `\\x[0-9a-f]{2}` | MEDIUM | Hex 인코딩 |
| 환경 접근 | `process.env` | LOW | 환경변수 접근 |
| 환경 접근 | `os.environ` | LOW | 환경변수 접근 |
| 자격증명 | `credential`, `keychain`, `password` | MEDIUM | 자격증명 접근 가능성 |

**package.json 특별 검사:**

```bash
gh api repos/OWNER/REPO/contents/package.json -H "Accept:application/vnd.github.raw" | \
  jq '.scripts | to_entries[] | select(.key | test("install|prepare|build")) | "\(.key): \(.value)"'
```

위험한 npm 스크립트:
- `preinstall` - npm install 전 실행
- `postinstall` - npm install 후 실행
- `prepare` - npm install 후 실행 (publish 전에도)

### Phase 6: 리포트 생성

수집한 정보를 종합하여 보안 리포트를 생성합니다.

**출력 형식:**

```markdown
## GitHub Repository Security Report

**Repository**: owner/repo
**Scan Date**: YYYY-MM-DD HH:MM

### Trust Indicators
| 지표 | 값 | 상태 |
|-----|-----|-----|
| Stars | 1,234 | [안전/주의/위험] |
| Forks | 456 | [안전/주의/위험] |
| Last Update | 3일 전 | [안전/주의/위험] |
| Owner Type | Organization | [안전/주의/위험] |
| License | MIT | [안전/주의/위험] |

### Risk Analysis

**CRITICAL:**
- [발견 시 나열]

**HIGH RISK:**
- [발견 시 나열]

**MEDIUM RISK:**
- [발견 시 나열]

### Flagged Files (Review Required)
1. 파일명 - 이유
2. 파일명 - 이유

### Recommendation
[SAFE / CAUTION / DANGER] - 권장 조치 설명
```

**최종 판정 기준:**

| 판정 | 조건 |
|------|------|
| SAFE | CRITICAL/HIGH 없음, 신뢰 지표 양호 |
| CAUTION | HIGH 1-2개 또는 MEDIUM 다수, 리뷰 권장 |
| DANGER | CRITICAL 존재 또는 HIGH 다수, 클론 비권장 |

## 추가 권장사항

CAUTION/DANGER 판정 시:

1. **격리 환경 테스트**: Docker 컨테이너에서 먼저 실행
   ```bash
   docker run --rm -it -v $(pwd):/app node:20 sh
   ```

2. **네트워크 모니터링**: 의심스러운 외부 통신 확인

3. **코드 리뷰**: 플래그된 파일 직접 검토

4. **대안 검색**: 더 신뢰할 수 있는 유사 패키지 검색

## 예시

```bash
# 사용 예시
/repo-safety-check https://github.com/expressjs/express
/repo-safety-check lodash/lodash
```

## 제한사항

- Private 저장소: gh CLI 인증 필요
- Rate Limit: GitHub API 제한 (인증 시 5000회/시간, 미인증 시 60회/시간)
- 대용량 저장소: 파일이 많은 경우 일부만 샘플링
