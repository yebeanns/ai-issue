# 작업 단계 2 (step2.md)

`docs/`에 작성한 문서 3건(TIL / 회고 / 부검)을 **Sprint 2 마일스톤 이슈로 등록**하고
검증하기까지 사용한 명령어 기록입니다. 1단계는 [`step.md`](./step.md) 참고.

- 저장소: <https://github.com/yebeanns/ai-issue>
- 대상 마일스톤: `Sprint 2 · 추론 & 회귀 분석` (#2)
- 생성 이슈: **#14, #15, #16**
- 작업 디렉터리: `C:\Users\LT-012\Desktop\hy-01\workspace\ai-issue`

---

## 0. 사전 확인

```bash
cd "C:/Users/LT-012/Desktop/hy-01/workspace/ai-issue"

# 문서 3건이 있는지 확인
ls -l docs/
# docs/postmortem.md  docs/retrospective.md  docs/til.md

# 줄 수 확인 (본문 요약에 사용)
wc -l docs/*.md
```

```bash
# 현재 이슈 / 마일스톤 상태 확인
gh issue list --repo yebeanns/ai-issue --state all --limit 50 --json number,title
gh api repos/yebeanns/ai-issue/milestones \
  --jq '.[] | "\(.number) \(.title) due=\(.due_on)"'
```

> 확인 결과: 마일스톤 `#2`가 `Sprint 2 · 추론 & 회귀 분석`, 기존 이슈는 `#1 ~ #13`.

---

## 1. 이슈 생성 방식 선택

문서 본문이 길고 한글이 많아, 1단계와 동일하게 **UTF-8 JSON + API** 방식을 사용했습니다.

### 방법 A. 이슈 1건씩 수동 생성 (`gh api --input`)

```bash
# payload 파일 예시 (UTF-8로 저장)
cat > /tmp/issue-14.json <<'JSON'
{
  "title": "[문서] TIL 14일 기록 정리 (2026-11-07 ~ 11-20)",
  "body": "## 목적\n...",
  "labels": ["documentation"],
  "milestone": 2
}
JSON

# 엔드포인트 선행 '/' 제거 (Git Bash 경로 변환 방지)
gh api --method POST repos/yebeanns/ai-issue/issues \
  --input /tmp/issue-14.json \
  --jq '{number:.number,title:.title,url:.html_url}'
```

### 방법 B. 3건 일괄 생성 (Python 스크립트)

건수가 여러 개이고 **멱등**(재실행 시 중복 생성 방지)이 필요해 스크립트로 처리했습니다.

```bash
PYTHONIOENCODING=utf-8 python \
  "C:/Users/LT-012/AppData/Local/Temp/opencode/create_doc_issues.py"
```

스크립트 동작:

1. `gh auth token`으로 토큰을 런타임에 읽음 (파일에 저장하지 않음)
2. 마일스톤 목록을 조회해 `Sprint 2 · 추론 & 회귀 분석` 번호 확인 → 없으면 즉시 중단
3. 기존 이슈 제목 집합을 조회 → 이미 있으면 `[skip]`
4. 없으면 `POST /repos/{owner}/{repo}/issues` 로 생성 (title/body/labels/milestone)
5. 마일스톤별 open/closed 이슈 수를 출력

사용한 API:

| 동작 | Method | Endpoint |
|---|---|---|
| 마일스톤 조회 | GET | `/repos/{owner}/{repo}/milestones?state=all&per_page=100` |
| 이슈 목록 조회 | GET | `/repos/{owner}/{repo}/issues?state=all&per_page=100` |
| 이슈 생성 | POST | `/repos/{owner}/{repo}/issues` |

실행 결과:

```text
[new ] issue #14: [문서] TIL 14일 기록 정리 (2026-11-07 ~ 11-20)
[new ] issue #15: [문서] 스프린트 회고 2회 (Sprint 1 / Sprint 2)
[new ] issue #16: [문서] 가상 장애 부검: v0.2.0 p-value underflow
milestone #2: Sprint 2 · 추론 & 회귀 분석 | open=9 closed=0
```

---

## 2. 이슈 본문 구성

| 이슈 | 문서 | 본문 구성 |
|---|---|---|
| #14 | `docs/til.md` | 목적, Day 1~7 / Day 8~14 표, 관련 이슈 번호, 체크리스트 |
| #15 | `docs/retrospective.md` | Why-What-How 요약, 스프린트 비교표, 액션 아이템 체크리스트 |
| #16 | `docs/postmortem.md` | STAR 요약, 근본 원인, 재발 방지 액션 5개, 관련 이슈 링크 |

공통 규칙:

- 제목 접두사 `[문서]`로 문서 작업임을 표시
- 라벨 `documentation` 고정
- 마일스톤은 `Sprint 2` (이미 완료된 문서 작업을 해당 스프린트 실적으로 기록)
- 본문에 산출물 경로(`docs/*.md`)와 관련 이슈 번호를 명시

---

## 3. 검증

```bash
# 이슈 + 마일스톤 + 라벨 한눈에 보기
gh issue list --repo yebeanns/ai-issue --state all --limit 50 \
  --json number,title,milestone,labels \
  --template '{{range .}}#{{.number}} {{.title}} | ms={{if .milestone}}{{.milestone.number}}{{else}}-{{end}} | {{range .labels}}{{.name}} {{end}}{{"\n"}}{{end}}'
```

```text
#16 [문서] 가상 장애 부검: v0.2.0 p-value underflow | ms=2 | documentation
#15 [문서] 스프린트 회고 2회 (Sprint 1 / Sprint 2)     | ms=2 | documentation
#14 [문서] TIL 14일 기록 정리 (2026-11-07 ~ 11-20)    | ms=2 | documentation
...
#1  배포됨                                            | ms=- |
```

```bash
# 문서 이슈만 필터링
gh issue list --repo yebeanns/ai-issue --label documentation --state all \
  --json number,title --jq '.[] | "#\(.number) \(.title)"'

# 마일스톤 진행률 확인
gh api repos/yebeanns/ai-issue/milestones \
  --jq '.[] | "#\(.number) \(.title) | open=\(.open_issues) closed=\(.closed_issues)"'
```

---

## 4. 로컬 문서 커밋 / 푸시

```bash
git status --porcelain
# ?? docs/
# ?? step.md
# ?? step2.md

git add docs/ step.md step2.md
git commit -m "docs: add TIL, sprint retrospectives, and postmortem"
git push
```

```bash
# 원격 반영 확인
git log --oneline -3
git status -sb
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `invalid API endpoint: "C:/Program Files/Git/repos/..."` | Git Bash가 `/repos/...`를 경로로 변환 | 엔드포인트 선행 `/` 제거 |
| 이슈 본문 한글 깨짐 | Windows 셸 인코딩 | UTF-8 JSON + `--input`, 스크립트는 `PYTHONIOENCODING=utf-8` |
| 같은 이슈가 중복 생성됨 | 멱등 처리 없음 | 생성 전 제목 목록 조회 후 `[skip]` |
| `milestone: 2` 지정이 무시됨 | 마일스톤 **번호**가 아닌 제목을 넣음 | `milestones` 조회로 번호를 먼저 확보 |
| 라벨이 붙지 않음 | 존재하지 않는 라벨 이름 | `GET /labels`로 존재 확인 (없으면 422) |
