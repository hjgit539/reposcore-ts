# 문서 빌드 도구 실행 환경 가이드

## 왜 Python이 필요한가?

이 프로젝트의 CLI 앱 본체는 TypeScript/Bun으로 작성되어 있습니다.
그러나 README와 docs 문서 목록을 생성하고 최신 상태인지 검증하는
문서 자동화 계층은 Python 기반 도구로 구현되어 있습니다.

구체적으로는 `tools/` 아래의 Python 스크립트가 데이터를 생성하고,
`jinja2-cli`가 그 데이터를 `.md` 템플릿에 끼워 넣어 최종 파일을 출력합니다.

따라서 문서를 수정한 뒤 `make docs` 또는 `make synopsis`로 결과물을 재생성하려면
Bun 외에 Python과 jinja2-cli가 설치되어 있어야 합니다.

---

## 설치 방법

프로젝트 루트의 `requirements.txt`에 필요한 의존성이 정의되어 있습니다.

```bash
pip install -r requirements.txt
```

`requirements.txt` 내용:

```
jinja2
jinja2-cli
```

---

## 각 도구의 역할

### tools/build-docs-data.py

`docs/` 폴더의 마크다운 파일을 탐색하여 문서 목록 데이터를 생성합니다.

- `README.md`, `README-template.md`를 제외한 `.md` 파일을 수집
- 각 파일의 첫 번째 H1(`#`) 헤더를 제목으로 추출
- 파일명 기준 알파벳 순으로 정렬
- `{"docs": [...]}` 형식의 JSON을 stdout으로 출력

이 데이터는 `docs/README-template.md`에 끼워 넣어 `docs/README.md`를 생성하는 데 사용됩니다.

### tools/build-synopsis-data.py

`bun run index.ts --help`를 실행하여 CLI 도움말 출력을 캡처합니다.

- `bun run index.ts --help` 실행 결과를 수집
- `{"synopsis": "<help 출력>"}` 형식의 JSON을 stdout으로 출력

이 데이터는 `README-template.md`에 끼워 넣어 `README.md`의 Synopsis 섹션을 생성하는 데 사용됩니다.

---

## make 명령과의 연결

`Makefile`은 다음과 같이 두 도구를 정의합니다.

```makefile
PYTHON ?= python3
JINJA2 ?= python3 -m jinja2cli
```

각 명령의 실행 흐름은 아래와 같습니다.

### make docs

`docs/README.md`를 재생성합니다.

```
build-docs-data.py  →  JSON 데이터 출력
    ↓ (파이프)
jinja2cli  →  docs/README-template.md에 데이터 삽입  →  docs/README.md 생성
```

### make synopsis

`README.md`의 Synopsis 섹션을 재생성합니다.

```
build-synopsis-data.py  →  bun run index.ts --help 캡처  →  JSON 데이터 출력
    ↓ (파이프)
jinja2cli  →  README-template.md에 데이터 삽입  →  README.md 생성
```

---

## 로컬 재생성 절차

### docs/ 문서를 수정한 경우

```bash
make docs
```

`docs/` 아래 `.md` 파일을 추가·삭제하거나 H1 제목을 변경한 경우 반드시 실행합니다.
실행 후 `docs/README.md`의 문서 목록이 자동으로 갱신됩니다.

### CLI 옵션이나 도움말을 수정한 경우

```bash
make synopsis
```

`index.ts`의 옵션, 인수, 도움말 출력이 변경된 경우 반드시 실행합니다.
실행 후 `README.md`의 Synopsis 섹션이 자동으로 갱신됩니다.

### 둘 다 갱신하는 경우

```bash
make
```

`make docs`와 `make synopsis`를 한 번에 실행합니다.

---

## CI 검증

다음 두 CI 워크플로우가 생성 결과물의 동기화 상태를 검증합니다.

| 워크플로우 | 검증 대상 |
|---|---|
| `docs-readme-validation` | `docs/README.md`가 최신인지 확인 |
| `docs-synopsis-validation` | `README.md`의 Synopsis가 최신인지 확인 |

각 워크플로우는 Python 설치 → jinja2-cli 설치 → `make` 실행 → `git diff`로 변경 여부를 확인하는 순서로 동작합니다.
로컬에서 `make`를 실행하지 않고 커밋하면 CI에서 실패할 수 있습니다.

---

## 참고
- [Jinja2 공식 문서](https://jinja.palletsprojects.com/)
- [jinja2-cli GitHub](https://github.com/mattrobenolt/jinja2-cli)
