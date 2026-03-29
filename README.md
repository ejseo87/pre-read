# Education Agent — 원서 예습노트 생성기

`main.ipynb`에 구현된 LangGraph 기반 교육용 에이전트이다.
사용자가 영어 원문과 학습 모드를 입력하면, 선호도 비율에 따라 **단어/표현 한국어 풀이**와 **문장 구조 한국어 해설**이 담긴 예습노트를 자동으로 생성한다.

**참고**

- 클럽 최종 미션 과제(preread)를 구현 중이어서, backend에 일부인 LangGraph 에이전트도 미완성이다.
- 클럽 최종 미션 과제(preread)의 설계서에 LangGraph부분을 참고하여 과제19를 수행한 것이다.
- 시간이 부족하여 codex의 손을 빌렸는데 코드 구조가 내 취향과 다르게 잡혔다.
- 막판에 codex가 shutdown되어서 claude code에게 긴급하게 도움을 요청해서 완성했다.
- 코드 생성은 claude code가 가장 잘한다.

---

## 그래프 구조

![[./과제19_그래프구조.png]]

`analyze_material` 이후 `focus_mode` 값에 따라 **Conditional Edge**가 분기한다.

**참고** 그래프 구조는 main.ipynb에도 표시되어 있다.

---

## 노드 설명

| 노드               | 역할                                                                                          |
| ------------------ | --------------------------------------------------------------------------------------------- |
| `ingest_material`  | 원문의 불필요한 공백을 정규화해 `cleaned_material`에 저장                                     |
| `analyze_material` | regex 기반 키워드 추출(빈도 상위 5개)과 앞 2문장 요약 생성                                    |
| `vocab_focus`      | 어휘 중심 액션 플랜 수립 + **ExampleBankTool** 호출로 주제별 예문 수집                        |
| `sentence_focus`   | 문장 구조 중심 액션 플랜 수립 + **ExampleBankTool** 호출로 주제별 예문 수집                   |
| `assemble_note`    | LLM을 2회 호출해 vocab/sentence 항목을 배치 생성하고, 선호도 비율을 적용해 최종 예습노트 조립 |

---

## 상태 (EducationState)

```python
# 입력
learner_name  : str
topic         : str그래
material      : str                        # 학습할 영어 원문
focus_mode    : Literal["vocab", "sentence"]

# 파이프라인이 채우는 중간 상태
cleaned_material  : str
keywords          : list[str]
summary           : str
examples          : list[str]              # ExampleBankTool 반환값
focus_plan        : dict[str, list[str]]   # 액션 플랜

# 최종 결과
vocab_items       : list[dict]  # {term, meaning, context_example}
sentence_items    : list[dict]  # {source_sentence, structure_breakdown, why_difficult}
preread_note      : str         # 완성된 마크다운 예습노트
preference_info   : dict        # 적용된 비율 메타데이터
memory_log        : list[str]   # 각 노드의 처리 내역
```

---

## 사용자 선호도 (focus_mode)

| 모드                   | 단어/표현 | 문장 구조 |
| ---------------------- | --------- | --------- |
| `vocab` (단어 위주)    | 70% — 7개 | 30% — 3개 |
| `sentence` (문장 위주) | 40% — 4개 | 60% — 6개 |

- **단어/표현 항목**: 문맥 속 한국어 의미 + 원문 예문
- **문장 구조 항목**: 원문 문장 + 주어/동사/절 구조 한국어 해설 + 어려운 이유

---

## Tool — ExampleBankTool

주제별 예문 데이터베이스에서 학습 키워드와 관련된 예문을 필터링해 반환하는 커스텀 툴이다.
지원 주제: `ai`, `productivity` (그 외는 `default` 풀 사용)

```python
example_tool(topic="productivity", focus_mode="vocab", keywords=["micro-brief", "momentum"])
# → ["Batching similar tasks reduces cognitive switching costs.", ...]
```

---

## LLM — PreReadNoteLLM

클럽 최종 미션 과제인 preread 프로젝트의 백엔드 `generate_preread_note_node`의 프롬프트 구조를 참고해 구현했다.

- `generate_vocab_items(topic, text, n)` → JSON 배열 1회 호출로 n개의 단어 항목 생성
- `generate_sentence_items(text, n)` → JSON 배열 1회 호출로 n개의 문장 항목 생성
- `OPENAI_API_KEY` 없으면 키워드/문장 분리 기반 fallback 플레이스홀더 반환

모델 설정:

```
OPENAI_MODEL=gpt-4o-mini  # 기본값, 환경변수로 변경 가능
```

---

## 실행 방법

```bash
pip install langgraph langchain-openai
export OPENAI_API_KEY=sk-...
```

노트북을 순서대로 실행하면 `vocab` / `sentence` 두 모드의 예습노트가 출력된다.

```python
result = education_agent.invoke({
    "learner_name": "Alice",
    "topic": "productivity",
    "focus_mode": "vocab",          # 또는 "sentence"
    "material": "원문 텍스트...",
    "memory_log": [],
})
print(result["preread_note"])
```

---

## 과제 요구사항 대응

| 요구사항         | 구현                                                                                          |
| ---------------- | --------------------------------------------------------------------------------------------- |
| 최소 3개 노드    | 5개 (`ingest_material`, `analyze_material`, `vocab_focus`, `sentence_focus`, `assemble_note`) |
| Conditional Edge | `route_focus` — `focus_mode` 값으로 vocab/sentence 경로 분기                                  |
| Tool 연동        | `ExampleBankTool` (커스텀)                                                                    |
| 메모리 기능      | `memory_log` — 각 노드 처리 내역 누적                                                         |
| LLM 연동         | `PreReadNoteLLM` — vocab/sentence 항목 배치 생성                                              |
