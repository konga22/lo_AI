# 플로터 로컬 AI 수업

Docker와 vLLM으로 개인 컴퓨터에서 로컬 AI 모델을 실행하고, 그 위에 Flutter 앱과 Supabase를 얹어 **내 문서를 근거로 답하는 RAG 노트 앱 NoteMind**를 만드는 수업 기록입니다.

- 1부 · 로컬 AI 서버: Docker, NVIDIA GPU, vLLM, 양자화, 모델 선택 → [2026-09-10 수업 정리](./class-notes-2026-09-10.md)
- 2부 · NoteMind RAG 앱: 아래 튜토리얼 → 자세한 기록은 [2026-10-08 · 10-15 수업 정리](./class-notes-2026-10-15.md)

---

## NoteMind 튜토리얼

PDF·텍스트 노트를 올리면 내용을 잘게 나눠 벡터로 저장하고, 질문하면 가장 관련 있는 부분을 찾아 그 근거로만 답하는 앱입니다. 모든 것이 내 PC에서 돌아갑니다.

```text
업로드: 파일 → 텍스트 추출 → 청크 분할 → 임베딩 → pgvector 저장
질문:   질문 → 임베딩 → 벡터 검색(근거) → 근거 + 질문 → 채팅 모델 → 스트리밍 답변 [1][2]
```

| 구성 요소 | 역할 | 주소 |
| --- | --- | --- |
| Supabase (Docker) | 익명 로그인, PostgreSQL + pgvector, 파일 Storage | `127.0.0.1:54321` |
| vLLM 채팅 서버 | 답변 생성 (`notemind-chat`) | `127.0.0.1:8000` |
| vLLM 임베딩 서버 | 문장 → 1024차원 벡터 (`notemind-embedding`) | `127.0.0.1:8001` |
| Flutter 앱 | 업로드·질문 화면 (Android 에뮬레이터) | `adb reverse`로 위 포트 연결 |

시작 코드는 수업 저장소 [scadasystems/notemind-rag-course](https://github.com/scadasystems/notemind-rag-course)의 `app-starter/`입니다. 화면과 상태 관리는 완성되어 있고 core 함수 12개(TODO 1~12)가 비어 있습니다. 이 저장소의 [`notemind-lab/`](./notemind-lab/)에는 **TODO 1~12를 모두 채운 코드**가 있습니다.

### 0. 준비물

| 항목 | 이 튜토리얼에서 쓴 것 |
| --- | --- |
| OS | Windows 11 + PowerShell |
| GPU | NVIDIA RTX 4060 (VRAM 8GB) |
| Docker | Docker Desktop (WSL2, GPU 사용 가능) |
| Flutter | 3.47.6 (Dart 3.13.5 이상 필요) |
| Android | SDK + JDK 21 + 에뮬레이터 (Android Studio는 없어도 됨) |
| Python | `init-env.py` 실행용 |

```powershell
git clone https://github.com/konga22/lo_AI
cd lo_AI/notemind-lab
```

### 1. 서버 준비 (처음 한 번)

**1-1. 비밀값 파일 만들기.** Supabase 키, DB 비밀번호, vLLM API 키가 랜덤으로 생성됩니다. `.env`는 `.gitignore`로 제외되므로 저장소에 올라가지 않습니다.

```powershell
python docker/init-env.py
```

**1-2. Supabase 시작과 DB 마이그레이션.**

```powershell
docker compose --project-directory docker/supabase up -d --wait
Get-Content docker/supabase/migrations/001_notemind.sql | docker compose --project-directory docker/supabase exec -T db psql -U postgres -d postgres -v ON_ERROR_STOP=1
```

마이그레이션이 만드는 것:

- 테이블 `documents · chunks · conversations · messages` (청크에 `vector(1024)` 칸)
- Storage 버킷 `documents`
- DB 함수 `finish_document`(청크를 한 번에 저장하고 문서를 `ready`로) · `match_chunks`(코사인 유사도 검색)
- RLS(행 단위 보안): 각 사용자는 자기 데이터만 읽고 쓸 수 있음

**1-3. vLLM 시작.** 채팅 서버가 healthy가 된 뒤 임베딩 서버가 뜹니다.

```powershell
docker compose --project-directory docker/vllm up -d --wait
```

`docker/vllm/.env`에서 모델을 고릅니다. 수업 기본값은 `Qwen/Qwen3-4B-AWQ`(`CHAT_GPU_MEMORY_UTILIZATION=0.65`)이고, 이 튜토리얼은 이미 받아 둔 `Qwen/Qwen2.5-7B-Instruct-GPTQ-Int4`(0.85)로 진행했습니다.

### 2. Android 에뮬레이터 준비 (Android Studio 없이)

```powershell
$env:JAVA_HOME = "C:\Program Files\Android\openjdk\jdk-21.0.8"
$sdk = "C:\Program Files (x86)\Android\android-sdk"
& "$sdk\cmdline-tools\latest\bin\sdkmanager.bat" --licenses
& "$sdk\cmdline-tools\latest\bin\sdkmanager.bat" "emulator" "system-images;android-35;google_apis;x86_64"
& "$sdk\cmdline-tools\latest\bin\avdmanager.bat" create avd -n notemind -k "system-images;android-35;google_apis;x86_64" -d pixel_7
```

Flutter는 **공백이 있는 SDK 경로를 지원하지 않습니다.** `Program Files (x86)`처럼 공백이 있으면 공백 없는 별칭(정션)을 만들어 알려 줍니다.

```powershell
cmd /c mklink /J C:\Android\sdk "C:\Program Files (x86)\Android\android-sdk"
flutter config --android-sdk C:\Android\sdk
flutter config --jdk-dir "C:\Program Files\Android\openjdk\jdk-21.0.8"
```

### 3. 앱 실행과 연결

**순서가 중요합니다.** 8GB GPU에서는 vLLM이 GPU 메모리를 먼저 잡아야 하므로, 에뮬레이터는 vLLM이 뜬 뒤 **GPU를 쓰지 않는 모드**로 켭니다.

```powershell
& "C:\Android\sdk\emulator\emulator.exe" -avd notemind -gpu swiftshader_indirect

# 에뮬레이터의 127.0.0.1을 PC의 서버로 연결 (에뮬레이터를 켤 때마다)
$adb = "C:\Android\sdk\platform-tools\adb.exe"
& $adb reverse tcp:54321 tcp:54321
& $adb reverse tcp:8000 tcp:8000
& $adb reverse tcp:8001 tcp:8001

cd app-starter
flutter pub get
flutter run
```

처음 실행하면 **연결 설정** 탭이 열립니다.

| 입력란 | 값 |
| --- | --- |
| Supabase 공개 키 | `docker/supabase/.env`의 `ANON_KEY` |
| 로컬 추론 API 키 | `docker/vllm/.env`의 `VLLM_API_KEY` |
| 주소·모델 ID | 기본값 그대로 |

**설정 저장** → **연결 확인**에서 `Supabase · 연결됨`, `생성 모델 · 연결됨`, `임베딩 모델 · 연결됨`이 나오면 준비 완료입니다.

### 4. TODO 구현

시작 코드의 빈 함수는 `UnimplementedError('TODO(n)')`를 던집니다. 아래 순서로 채우며, 각 단계의 테스트로 확인합니다. 전체 코드는 링크된 파일에 있습니다.

| 파트 | TODO | 파일 |
| --- | --- | --- |
| 업로드 | 1 `chunkText` · 2 `extractDocumentChunks` · 3 `embed` · 4 `ingest` · 5 `retryDocument` | [text_processing.dart](./notemind-lab/app-starter/lib/core/text_processing.dart) · [document_import.dart](./notemind-lab/app-starter/lib/core/document_import.dart) · [inference.dart](./notemind-lab/app-starter/lib/data/inference.dart) · [repository.dart](./notemind-lab/app-starter/lib/data/repository.dart) |
| 질문 | 6 `createConversation` · 7 `saveMessage` · 8 `isDocumentOverviewRequest` · 9 `search` · 10 `overview` · 11 `sseData` · 12 `chat` | 위와 같음 |

#### Step 1 · 텍스트를 청크로 자르기 (TODO 1)

임베딩 모델이 다루기 좋은 크기(320자)로 자르되, 경계에서 문맥이 끊기지 않게 48자씩 겹칩니다. 문자열이 아니라 **`runes`(유니코드 코드포인트)** 단위로 잘라야 한글·이모지가 깨지지 않습니다.

```dart
final runes = text.replaceAll('\r\n', '\n').replaceAll('\u0000', '').trim().runes.toList();
for (var start = 0; start < runes.length; start += size - overlap) {
  final end = (start + size).clamp(0, runes.length);
  final chunk = String.fromCharCodes(runes.sublist(start, end)).trim();
  if (chunk.isNotEmpty) chunks.add(chunk);
  if (end == runes.length) break; // 끝부분이 중복 청크로 들어가지 않게
}
```

```powershell
flutter test test/core_test.dart --plain-name chunks
```

#### Step 2 · 파일에서 텍스트 뽑기 (TODO 2)

- **txt·md**: UTF-8로 읽고 맨 앞 BOM(`\uFEFF`)을 지운 뒤 `chunkText`
- **PDF**: `pdfrx`로 페이지마다 텍스트를 뽑고, 청크 앞에 `[PDF n페이지]`를 붙여 답변에서 출처 페이지를 보여 줄 수 있게 합니다.
- 스캔 PDF(텍스트 0개) → OCR 안내, 암호 PDF → 암호 안내, 500페이지 초과 → 거부. `finally`에서 문서를 반드시 닫습니다.

```dart
for (final page in pages) {
  final text = (await page.loadStructuredText()).fullText;
  chunks.addAll(chunkText(text).map((part) => '[PDF ${page.pageNumber}페이지]\n$part'));
}
```

```powershell
flutter test test/document_import_test.dart
```

#### Step 3 · 임베딩 (TODO 3)

`POST /v1/embeddings`로 문장 목록을 보내 1024차원 벡터를 받습니다. 응답 순서가 섞일 수 있으므로 **`index`로 다시 정렬**하지 않으면 문장과 벡터가 엇갈려 저장됩니다. 차원이 1024가 아니면 DB 칸(`vector(1024)`)에 맞지 않으므로 거부합니다.

```dart
final data = (jsonDecode(response.body)['data'] as List)
    .map((e) => Map<String, dynamic>.from(e)).toList()
  ..sort((a, b) => (a['index'] as int).compareTo(b['index'] as int));
```

```powershell
flutter test test/core_test.dart --plain-name embedding
```

#### Step 4 · 업로드와 색인 (TODO 4 · 5)

1. 청크 만들기 → `documents`에 기록(새 문서 insert, 재시도는 `processing`으로 update)
2. Storage에 원본 업로드
3. 청크를 **4개씩** 임베딩하며 진행률 표시
4. `rpc('finish_document')`로 청크를 **한 번에** 저장 → 반쯤 저장된 문서가 검색에 섞이지 않음
5. 실패하면 문서를 `failed`로 표시하고 원래 오류를 다시 던짐

재시도(TODO 5)는 Storage에서 원본을 내려받아 같은 id로 `ingest(retry: document)`를 다시 부릅니다.

> 여기까지 하면 **문서함 → 노트 업로드**가 끝까지 동작합니다. 카드에 `검색 준비 완료 · N개 청크`가 뜹니다.

#### Step 5 · 대화와 메시지 저장 (TODO 6 · 7)

대화는 새로 만드는 것이라 `insert`, 메시지는 답변이 스트리밍되는 동안 `generating → complete`로 상태가 바뀔 때마다 같은 id로 다시 저장하므로 `upsert`입니다.

#### Step 6 · 요약 요청 판별 (TODO 8)

"요약해줘"처럼 **주제가 없는** 요청은 벡터 검색으로 맞는 근거를 찾을 수 없으므로 문서 앞부분을 근거로 씁니다. "화성의 기온을 요약해줘"처럼 주제가 붙으면 검색해야 합니다. 정규식 `^…$`로 **문장 전체**가 요약 요청 틀과 같은지 봅니다.

```text
^ [이 자료·문서·노트·PDF] [전체·핵심 내용을] (요약|정리) (해줘|해 주세요) $
   앞: 생략 가능            중간: 생략 가능      꼭 있어야 함   끝
```

```powershell
flutter test test/core_test.dart --plain-name overview
```

#### Step 7 · 근거 찾기 (TODO 9 · 10)

**검색(TODO 9)** — 질문 벡터로 DB 함수 `match_chunks`를 호출합니다. RAG의 R(Retrieval)입니다.

```dart
await client.rpc('match_chunks', params: {
  'query_embedding': vector,
  'p_model': model,            // 같은 임베딩 모델로 만든 문서만
  'match_threshold': 0.45,     // 코사인 유사도 0.45 이상
  'match_count': 5,            // 최대 5개
  'p_document_id': documentId, // "이 노트에 질문하기"면 그 문서만
});
```

**요약용 근거(TODO 10)** — 문서를 하나만 골라 `position` 순으로 9개를 가져와 앞 8개를 씁니다. 9번째가 있으면 문서가 더 길다는 뜻이므로 `partial: true`로 표시합니다.

```powershell
flutter test test/overview_repository_test.dart
```

#### Step 8 · 스트리밍 답변 (TODO 11 · 12)

채팅 서버는 답변을 토큰마다 SSE 형식으로 흘려보냅니다.

```text
data: {"choices":[{"delta":{"content":"안"}}]}

data: {"choices":[{"delta":{"content":"녕"}}]}

data: [DONE]
```

**SSE 파싱(TODO 11)** — `utf8.decoder`(쪼개진 한글 바이트를 이어 붙임) → `LineSplitter` → `data:` 줄을 모아 **빈 줄마다** 이벤트 하나로 내보냅니다.

**chat(TODO 12)** — `http.Request`를 직접 만들어 `send()`하면 응답이 끝나기 전부터 `response.stream`으로 읽을 수 있습니다.

```dart
..body = jsonEncode({
  'model': config.model,
  'messages': messages,                                // system(규칙 + 근거 [1][2]…) + user
  'stream': true,
  'max_tokens': 768,
  'temperature': 0.2,                                  // 근거에 충실하게
  'chat_template_kwargs': {'enable_thinking': false},  // Qwen3 생각 모드 끄기
});
```

토큰은 받는 대로 `yield`하고, 마지막에 **`[DONE]`을 받았는지 확인**합니다. 받지 못했다면 연결이 중간에 끊긴 것이므로 잘린 답을 정상 답처럼 보여 주지 않도록 오류를 냅니다.

```powershell
flutter test   # 전체 13개
```

### 5. 직접 확인하기

에뮬레이터의 다운로드 폴더에 테스트할 PDF를 넣고 앱에서 업로드합니다. 아래 질문 예시는 수업 자료 `NoteMind-Earth-Space.pdf`(지구와 우주 심화 읽기, 6쪽) 기준입니다. 수업 자료는 이 저장소에 포함하지 않았습니다.

```powershell
& $adb push NoteMind-Earth-Space.pdf /sdcard/Download/
```

**문서함 → + 노트 업로드 → Downloads**에서 파일을 고르고, 준비가 끝나면 **이 노트에 질문하기**를 누릅니다.

| 질문 | 기대하는 동작 |
| --- | --- |
| 외핵을 S파가 통과하지 못하는 이유는? | 벡터 검색으로 1페이지 근거를 찾아 `[1]`을 달아 답함 |
| 이 문서를 요약해줘 | 요약 요청으로 판별 → 검색 없이 문서 앞부분으로 요약 |
| 관측 실습 제출 마감은 언제야? | 개정 이력의 옛 값에 속지 않고 11월 18일 18:00 |
| 담당 교사 연락처 알려줘 | 문서에 없으므로 "자료에 없습니다." |

에뮬레이터 통합 테스트(업로드 → 재시도 → RAG 답변)는 키를 넘겨 실행합니다. 앱이 다시 설치되므로 테스트 후 연결 설정을 다시 입력합니다.

```powershell
flutter test integration_test/pdf_test.dart -d emulator-5554 `
  --dart-define=SUPABASE_ANON_KEY=<ANON_KEY> `
  --dart-define=INFERENCE_API_KEY=<VLLM_API_KEY>
```

**현재 결과**: `flutter analyze` 문제 없음 · 단위 테스트 13개 통과 · `pdf_test` 통과 · `app_test`는 답(“미래관 402호”)은 맞지만 Qwen2.5-7B가 `[1]` 인용을 붙이지 않아 1개 항목 실패(수업 기본 모델 Qwen3-4B-AWQ에서는 통과할 가능성이 높음).

### 6. 문제 해결

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| `sdkmanager`가 아무것도 설치하지 않음 | 라이선스 동의 입력이 전달되지 않음 | `--licenses`로 먼저 동의 |
| `flutter devices`에 에뮬레이터 없음 | SDK 경로에 공백 | `C:\Android\sdk` 정션 + `flutter config --android-sdk` |
| `requires SDK version ^3.13.5` | Flutter의 Dart가 낮음 | `flutter upgrade` |
| 첫 빌드가 매우 느림 | Gradle·NDK(약 2GB) 다운로드 | 첫 빌드만 느림 |
| vLLM `No available memory for the cache blocks` | 8GB GPU에서 다른 프로그램(에뮬레이터 등)이 메모리를 씀, 또는 임베딩 서버가 먼저 뜸 | 채팅 → 임베딩 순서로 vLLM을 먼저 띄우고, 에뮬레이터는 `-gpu swiftshader_indirect`로 나중에 |
| `Free memory on device ... less than desired` | `gpu-memory-utilization`이 실제 여유 메모리보다 큼 | 비율을 올리지 말고 다른 GPU 사용을 줄임 |
| 앱에 `No API key found in request` | 연결 설정의 Supabase 키가 비어 있음 | 키를 넣고 설정 저장 |
| 통합 테스트 후 연결 설정이 비어 있음 | 테스트가 앱을 다시 설치 | 키 다시 입력 |

### 7. 종료

```powershell
docker compose --project-directory docker/vllm down
docker compose --project-directory docker/supabase down   # 데이터는 유지
```

`down -v`는 DB와 업로드 파일이 담긴 볼륨까지 지우므로 일반 종료에 쓰지 않습니다.

---

## 수업 기록

- [2026-09-10 · Docker, GPU, vLLM, 양자화, llmfit](./class-notes-2026-09-10.md)
- [2026-10-08 · 10-15 · NoteMind RAG 앱](./class-notes-2026-10-15.md)

## 보안 주의사항

- `notemind-lab/docker/**/.env`의 Supabase 키·DB 비밀번호·vLLM API 키는 `.gitignore`로 제외되어 저장소에 올라가지 않습니다. 각자 `python docker/init-env.py`로 생성합니다.
- Hugging Face 토큰은 비밀번호처럼 취급합니다. 실제 토큰을 소스 코드, 터미널 캡처, 문서 또는 GitHub 저장소에 기록하지 않습니다. 공용 컴퓨터에서는 수업 후 토큰을 폐기하고 모든 서비스에서 로그아웃합니다.

## 라이선스

`notemind-lab/`의 시작 코드는 [scadasystems/notemind-rag-course](https://github.com/scadasystems/notemind-rag-course)에서 가져왔으며 [MIT 라이선스](./notemind-lab/LICENSE)를 따릅니다. `notemind-lab/docker/supabase/`의 Supabase 구성 파일은 원본 라이선스([LICENSE.upstream](./notemind-lab/docker/supabase/LICENSE.upstream))를 따릅니다.
