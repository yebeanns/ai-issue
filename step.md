# 작업 단계 (step.md)

`ai-issue` 프로젝트를 GitHub 원격 저장소로 만들고, 통계 프로젝트용 라벨·마일스톤·예시 이슈를
반영하기까지 사용한 명령어를 순서대로 정리한 기록입니다.

- 저장소: <https://github.com/yebeanns/ai-issue>
- 계정: `yebeanns`
- 작업 디렉터리: `C:\Users\LT-012\Desktop\hy-01\workspace\ai-issue`
- 환경: Windows + Git Bash + GitHub CLI 2.103.3 + Python 3.10

---

## 0. 사전 확인

```bash
# gh 설치/버전 확인
gh --version

# 현재 폴더 기준 git 상태 (브랜치, 원격, 변경 파일)
git status --short
git remote -v
git branch --show-current

# 이 폴더가 속한 git 저장소 루트 확인
git rev-parse --show-toplevel
```

> 이때 `ai-issue`는 빈 폴더였고, git 저장소 루트는 상위 `workspace` 폴더였습니다.
> → `ai-issue`를 독립 저장소로 새로 초기화하기로 결정.

---

## 1. GitHub 인증

```bash
# 로그인 상태 확인
gh auth status

# 인증된 계정 확인
gh api user --jq .login
```

로그인이 안 되어 있으면 대화형 로그인을 먼저 수행합니다(브라우저 인증).

```bash
gh auth login
# GitHub.com → HTTPS → Login with a web browser
```

스크립트에서 토큰이 필요할 때는 아래처럼 가져옵니다(토큰을 파일/코드에 직접 저장하지 않음).

```bash
gh auth token
```

필요 권한(scopes): `repo` (이슈·라벨·마일스톤 생성), `read:org`

---

## 2. 로컬 저장소 준비

```bash
cd "C:/Users/LT-012/Desktop/hy-01/workspace/ai-issue"

# git 사용자 정보 확인 (커밋 author)
git config --get user.name
git config --get user.email

# 기본 브랜치를 main으로 하여 저장소 초기화
git init -b main
```

첫 커밋용 `README.md`를 만든 뒤:

```bash
git add README.md
git commit -m "chore: initial commit"
git log --oneline
```

---

## 3. 원격 저장소 생성 + 첫 push

```bash
# 같은 이름의 원격 레포가 이미 있는지 확인 (없으면 GraphQL 404)
gh repo view yebeanns/ai-issue --json nameWithOwner

# 원격 레포 생성 + origin 등록 + 현재 브랜치 push 를 한 번에
gh repo create ai-issue --public --source=. --remote=origin --push \
  --description "AI issue tracking"
```

```bash
# 결과 확인
git remote -v
git status -sb
gh repo view yebeanns/ai-issue \
  --json nameWithOwner,visibility,url,defaultBranchRef \
  --jq '{repo:.nameWithOwner,visibility:.visibility,url:.url,branch:.defaultBranchRef.name}'
```

---

## 4. 이슈 생성 (단건)

기존 이슈 확인:

```bash
gh issue list --repo yebeanns/ai-issue --state all --json number,title
```

한글 제목/본문은 Windows 셸 인코딩 문제를 피하기 위해 **UTF-8 JSON 파일 + `--input`** 방식을 사용했습니다.

```json
// issue.json
{
  "title": "배포됨",
  "body": "배포가 완료되었습니다."
}
```

```bash
gh api --method POST repos/yebeanns/ai-issue/issues \
  --input "C:/Users/LT-012/AppData/Local/Temp/opencode/issue.json" \
  --jq '{number:.number,title:.title,url:.html_url,state:.state}'
```

> ⚠️ Git Bash에서 `--input` 경로가 `/c/...` 형태로 변환되면
> `invalid API endpoint: "C:/Program Files/Git/..."` 오류가 납니다.
> **엔드포인트 앞의 `/`를 빼고** `repos/{owner}/{repo}/...` 형태로 쓰면 해결됩니다.

---

## 5. 라벨 · 마일스톤 · 예시 이슈 일괄 생성

건수가 많아 Python 스크립트(`setup_stats.py`)로 GitHub REST API를 호출했습니다.
토큰은 `gh auth token`으로 런타임에 읽고, 인증은 `urllib.request` 헤더로 전달합니다.

```bash
PYTHONIOENCODING=utf-8 python \
  "C:/Users/LT-012/AppData/Local/Temp/opencode/setup_stats.py"
```

스크립트가 사용한 API 엔드포인트:

| 동작 | Method | Endpoint |
|---|---|---|
| 라벨 목록 조회 | GET | `/repos/{owner}/{repo}/labels?per_page=100` |
| 라벨 생성 | POST | `/repos/{owner}/{repo}/labels` |
| 마일스톤 목록 조회 | GET | `/repos/{owner}/{repo}/milestones?state=all&per_page=100` |
| 마일스톤 생성 | POST | `/repos/{owner}/{repo}/milestones` |
| 이슈 목록 조회 | GET | `/repos/{owner}/{repo}/issues?state=all&per_page=100` |
| 이슈 생성 | POST | `/repos/{owner}/{repo}/issues` |

동작 방식:

1. 기존 라벨/마일스톤/이슈 이름을 먼저 조회
2. 이미 있으면 `[skip]` (멱등 실행 → 재실행해도 중복 생성 안 됨)
3. 없으면 생성 후 `[new ]` 로그 출력
4. 마지막에 라벨 수, 마일스톤별 open 이슈 수, 전체 이슈 수 요약

생성 결과:

- 라벨 12개 추가 (총 22개)
- 마일스톤 2개 (`Sprint 1` ~ 2026-11-13, `Sprint 2` ~ 2026-11-20)
- 이슈 12개 (`#2` ~ `#13`)

---

## 6. 검증

```bash
# 전체 이슈 수 (PR 제외 포함)
gh api "repos/yebeanns/ai-issue/issues?state=all&per_page=100" --jq 'length'

# 이슈 + 마일스톤 + 라벨을 한눈에 보기
gh issue list --repo yebeanns/ai-issue --state all --limit 50 \
  --json number,title,milestone,labels \
  --template '{{range .}}#{{.number}} {{.title}} | ms={{if .milestone}}{{.milestone.title}}{{else}}-{{end}} | {{range .labels}}{{.name}} {{end}}{{"\n"}}{{end}}'

# 라벨 이름만 확인
gh api repos/yebeanns/ai-issue/labels --jq '.[].name'
```

---

## 7. 참고: 자주 쓴 보조 명령

```bash
# 커밋할 변경 확인
git status --porcelain

# 변경 사항 커밋 + push
git add -A && git commit -m "docs: add step.md"
git push
```

---

## 트러블슈팅 정리

| 증상 | 원인 | 해결 |
|---|---|---|
| `You are not logged into any GitHub hosts` | gh 미인증 | `gh auth login` |
| `invalid API endpoint: "C:/Program Files/Git/..."` | Git Bash가 `/repos/...`를 경로로 변환 | 엔드포인트 선행 `/` 제거 |
| 한글 제목/본문 깨짐 | Windows 셸 인코딩 | UTF-8 JSON 파일 + `--input`, 스크립트는 `PYTHONIOENCODING=utf-8` |
| `gh repo create` 실패 (이미 존재) | 동일 이름 레포 존재 | `gh repo view`로 확인 후 `--remote`만 연결하거나 이름 변경 |
| 마일스톤 기한이 `00:00:00Z`로 표시 | GitHub `due_on`은 날짜만 반영 | 날짜 기준으로 계획 (KST 09:00) |
