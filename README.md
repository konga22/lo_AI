# 플로터 로컬 AI 수업

Docker 컨테이너와 vLLM을 이용해 개인 컴퓨터에서 로컬 AI 모델을 실행하고, OpenAI 호환 API로 사용하는 방법을 학습하는 프로젝트입니다.

## 수업에서 배우는 내용

- Docker 이미지·컨테이너·포트·볼륨의 기본 개념
- NVIDIA GPU가 Docker 컨테이너에 연결되는지 진단하는 방법
- vLLM으로 Hugging Face 모델을 OpenAI 호환 API 서버로 실행하는 방법
- 양자화와 GPU/RAM 용량을 고려해 실행 가능한 모델을 고르는 방법
- `llmfit`으로 하드웨어를 진단하고 모델 적합도와 속도를 비교하는 방법
- Windows/NVIDIA CUDA와 Apple Silicon/Metal 환경의 차이
- 로그를 읽고 CUDA, 메모리, 권한, 포트 오류를 해결하는 방법

## 전체 학습 흐름

1. `docker version`과 `nvidia-smi`로 실행 환경을 확인합니다.
2. CUDA 테스트 컨테이너로 GPU 연결 상태를 확인합니다.
3. `llmfit doctor`와 `llmfit fit`으로 장비에 맞는 모델을 고릅니다.
4. Docker 또는 vLLM Metal로 모델 서버를 실행합니다.
5. `/v1/models`와 `/v1/chat/completions`를 호출해 서버를 검증합니다.
6. Python OpenAI SDK의 `base_url`을 로컬 서버로 바꿔 응답을 확인합니다.
7. 벤치마크 결과를 바탕으로 모델을 다시 선택하고 반복합니다.

## NoteMind RAG 앱 실습

로컬 vLLM 위에 Flutter 앱과 Supabase를 얹어, 내 문서를 근거로 답하는 RAG 노트 앱 **NoteMind**를 만듭니다. 수업 저장소 [scadasystems/notemind-rag-course](https://github.com/scadasystems/notemind-rag-course)의 시작 코드에서 비어 있는 core 함수 12개(TODO)를 채웁니다.

- 코드: [`notemind-lab/`](./notemind-lab/) (`app-starter/` Flutter 앱, `docker/` Supabase·vLLM, `start.py`)
- 진행: TODO 1~10 완료 (문서 업로드 동작), TODO 11~12 남음
- 실행 순서, TODO별 설명, 문제 해결: [2026-10-08 · 10-15 수업 정리](./class-notes-2026-10-15.md)

```text
업로드: 파일 → 텍스트 추출 → 청크 분할 → 임베딩 → pgvector 저장
질문:   질문 → 임베딩 → 벡터 검색(근거) → 근거 + 질문 → 채팅 모델 → 스트리밍 답변
```

## 수업 기록

- [2026-09-10 수업 정리](./class-notes-2026-09-10.md)
- [2026-10-08 · 10-15 NoteMind RAG 앱 수업 정리](./class-notes-2026-10-15.md)

## 보안 주의사항

`notemind-lab/docker/**/.env`의 Supabase 키와 vLLM API 키는 `.gitignore`로 제외되어 저장소에 올라가지 않습니다. 키가 필요하면 각자 `python docker/init-env.py`로 생성합니다.

Hugging Face 토큰은 비밀번호처럼 취급합니다. 실제 토큰을 소스 코드, 터미널 캡처, 문서 또는 GitHub 저장소에 기록하지 않습니다. 공용 컴퓨터에서는 수업 후 토큰을 폐기하고 모든 서비스에서 로그아웃합니다.
