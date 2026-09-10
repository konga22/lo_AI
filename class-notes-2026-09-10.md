# 2026-09-10 로컬 AI 수업 정리

## 1. 오늘의 목표

Docker의 기본 사용법을 익히고, NVIDIA GPU를 컨테이너에서 사용할 수 있는지 확인했습니다. 이어서 vLLM, 양자화, Hugging Face 토큰, `llmfit`의 역할과 다음 실습 흐름을 학습했습니다.

## 2. 학교 PC 환경

- 운영체제: Windows
- 터미널: Windows PowerShell
- GPU: NVIDIA GeForce RTX 4060
- VRAM: 8,188 MiB(약 8GB)
- NVIDIA 드라이버: 560.94
- `nvidia-smi` 표시 CUDA 버전: 12.6
- Docker Engine: 29.7.2
- Docker Desktop: 4.90.0
- 컨테이너 환경: Linux (`desktop-linux`)

> `nvidia-smi`의 CUDA 버전은 현재 드라이버가 지원하는 최대 CUDA 계열을 뜻합니다. PC에 같은 버전의 CUDA Toolkit이 설치됐다는 뜻은 아닙니다.

## 3. Docker 기초 실습

### Docker 상태 확인

```powershell
docker version
```

클라이언트와 Linux 엔진 정보가 모두 출력되어 Docker Desktop이 정상 작동함을 확인했습니다.

### Nginx 컨테이너 실행

```powershell
docker run -d --name web -p 8080:80 nginx
```

- `-d`: 백그라운드 실행
- `--name web`: 컨테이너 이름을 `web`으로 지정
- `-p 8080:80`: PC의 8080번 포트를 컨테이너의 80번 포트에 연결
- `nginx`: 사용할 이미지 이름

실행 상태 확인:

```powershell
docker ps
```

컨테이너 종료와 삭제:

```powershell
docker stop web
docker rm web
```

### 발생했던 PowerShell 오류

```powershell
docker stop web && docker rm web
```

학교 PC의 Windows PowerShell에서는 `&&`가 지원되지 않아 오류가 발생했습니다. 두 명령을 한 줄씩 실행해 해결했습니다. Bash/macOS에서 사용하는 `\` 역시 PowerShell의 줄 연결 문자가 아닙니다. PowerShell에서는 줄 마지막에 백틱 문자를 사용하며, 백틱 뒤에 공백을 넣지 않습니다.

## 4. Docker에서 NVIDIA GPU 확인

호스트에서 GPU 확인:

```powershell
nvidia-smi
```

컨테이너에서 GPU 확인:

```powershell
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu22.04 nvidia-smi
```

컨테이너 안에서도 RTX 4060 정보가 출력되어 Docker GPU 연결 테스트를 통과했습니다.

## 5. vLLM 핵심 개념

vLLM은 로컬 모델을 OpenAI 호환 API 서버로 제공합니다. 기존 OpenAI SDK 코드도 주로 `base_url`만 로컬 서버 주소로 변경해 재사용할 수 있습니다.

| 주소 | 용도 |
| --- | --- |
| `/v1/models` | 서버에 올라온 모델 목록 |
| `/v1/completions` | 텍스트 이어쓰기 |
| `/v1/chat/completions` | 대화형 응답 생성 |
| `/health` | 서버 상태 확인 |
| `/metrics` | 처리량과 지연시간 지표 |

기본 포트는 `8000`입니다.

- PagedAttention: KV 캐시를 페이지 단위로 관리해 GPU 메모리를 효율적으로 사용합니다.
- Continuous batching: 들어오는 요청을 연속적으로 묶어 GPU 활용도를 높입니다.

## 6. 양자화

양자화는 모델 가중치의 정밀도를 낮춰 메모리 사용량을 줄이는 방법입니다.

| 정밀도 | 가중치당 크기 | 특징 |
| --- | ---: | --- |
| FP16/BF16 | 2바이트 | 품질이 높지만 메모리 사용량이 큼 |
| 8비트 | 1바이트 | 메모리를 절반 수준으로 줄이고 약간의 품질 손실 가능 |
| 4비트 | 0.5바이트 | 메모리를 4분의 1 수준으로 줄이지만 품질 손실 가능성이 더 큼 |

- AWQ, GPTQ: 주로 GPU용 4비트 양자화
- GGUF: CPU 친화적 형식
- EXL2: 가변 비트 양자화 형식

## 7. Hugging Face 준비

다음 수업 전까지 Hugging Face 계정을 만들고 개인 액세스 토큰을 준비합니다.

- 토큰 유형: `Read`
- 권한: 모델 다운로드에 필요한 읽기 권한만 사용
- 토큰 이름 예시: `lo-ai-class-school-temp`
- 실제 토큰은 문서나 GitHub에 기록하지 않음
- 공용 PC에서는 수업 후 토큰을 폐기하고 로그아웃

PowerShell 세션에서 토큰을 화면에 표시하지 않고 입력하는 예:

```powershell
$env:HF_TOKEN = (New-Object System.Net.NetworkCredential("", (Read-Host "Hugging Face 토큰 입력" -AsSecureString))).Password
```

사용 후 세션에서 제거:

```powershell
$env:HF_TOKEN = $null
```

## 8. Windows에서 vLLM 실행

### 수업에서 소개된 기본 형태

```powershell
docker run --rm --gpus all `
  -v "${env:USERPROFILE}\.cache\huggingface:/root/.cache/huggingface" `
  --env "HF_TOKEN=$env:HF_TOKEN" `
  -p 8000:8000 `
  --ipc=host `
  vllm/vllm-openai:latest `
  --model Qwen/Qwen2.5-1.5B-Instruct
```

처음 실행할 때 이미지와 모델을 내려받습니다. `Application startup complete`가 표시되면 서버가 준비된 것입니다. 종료는 `Ctrl+C`입니다.

### 학교 PC에서 발생한 CUDA 버전 충돌

실제 오류:

```text
nvidia-container-cli: requirement error: unsatisfied condition: cuda>=13.0
```

원인:

- `vllm/vllm-openai:latest`가 현재 CUDA 13.0을 요구함
- 학교 PC의 드라이버 560.94는 CUDA 12.6 계열까지만 지원함
- 이미지 다운로드에는 성공했지만 컨테이너 시작 단계에서 실패함

#### 해결 방법 A: 수업에서 지정한 CUDA 12 호환 태그 사용(권장)

강사가 지정한 버전이 있다면 그 태그를 가장 먼저 사용합니다. 현재 PC에서 시험할 수 있는 CUDA 12.1 기반 예시는 `v0.6.6`입니다.

```powershell
docker run --rm --gpus all `
  -v "${env:USERPROFILE}\.cache\huggingface:/root/.cache/huggingface" `
  --env "HF_TOKEN=$env:HF_TOKEN" `
  -p 8000:8000 `
  --ipc=host `
  vllm/vllm-openai:v0.6.6 `
  --model Qwen/Qwen2.5-1.5B-Instruct
```

> 오래된 vLLM 버전에서는 최신 모델 기능이 일부 지원되지 않을 수 있습니다. 수업에서 제공하는 고정 태그가 있으면 반드시 그 버전을 우선합니다.

#### 해결 방법 B: NVIDIA 드라이버 업데이트

CUDA 13.x 컨테이너는 580 이상 드라이버가 필요합니다. 학교 소유 PC에서는 강사 또는 관리자 승인을 받은 뒤 업데이트해야 합니다. Docker 실행을 위해 Windows에 CUDA Toolkit을 별도로 설치할 필요는 없으며, 호스트 NVIDIA 드라이버만 조건을 충족하면 됩니다.

## 9. Apple Silicon MacBook에서 실행

`vllm-metal`은 Apple Silicon과 Metal을 사용하는 커뮤니티 유지보수 플러그인입니다. 현재 사전 빌드 설치는 macOS 15 이상을 요구합니다. Intel Mac에서는 사용할 수 없습니다.

설치:

```bash
curl -fsSL https://raw.githubusercontent.com/vllm-project/vllm-metal/main/install.sh | bash
```

가상환경 활성화:

```bash
source ~/.venv-vllm-metal/bin/activate
```

4비트 MLX 모델 실행 예:

```bash
vllm serve "mlx-community/Meta-Llama-3.1-8B-Instruct-4bit" \
  --gpu-memory-utilization 0.9
```

메모리가 부족하면 `--gpu-memory-utilization 0.8`처럼 비율을 낮추거나 더 작은 모델을 선택합니다. 가상환경 종료 명령은 `deactivate`입니다. iTerm2는 선택 사항이며 macOS 기본 터미널에서도 같은 명령을 사용할 수 있습니다.

## 10. llmfit으로 모델 선택

`llmfit`은 CPU, RAM, GPU, VRAM을 읽어 장비에 맞는 모델과 예상 속도를 추천합니다.

주요 명령:

```text
llmfit doctor
llmfit
llmfit recommend
llmfit fit
llmfit info "모델명"
llmfit bench
llmfit recommend --json
```

설치 예:

```bash
# macOS/Linux (Homebrew)
brew install AlexsJones/llmfit/llmfit
```

```powershell
# Windows (Scoop이 설치된 경우)
scoop install llmfit
```

학교 PC에서는 PowerShell이 관리자 권한으로만 열려 Scoop 기본 설치가 차단되었습니다. 관리자 설치가 수업 정책상 허용된 경우 공식 설치 스크립트의 `-RunAsAdmin` 옵션을 사용합니다.

```powershell
irm get.scoop.sh -outfile install.ps1
./install.ps1 -RunAsAdmin
scoop install llmfit
```

`llmfit`을 실행한 뒤 방향키로 모델을 고르고 `Enter`를 누르면 상세 화면과 목록 화면을 전환할 수 있습니다.

## 11. 자주 발생하는 오류

| 오류 | 원인 | 해결 방향 |
| --- | --- | --- |
| `CUDA out of memory` | VRAM 부족 | 작은/양자화 모델, `--max-model-len 2048`, `--gpu-memory-utilization 0.85` |
| `could not select device driver ... [[gpu]]` | Docker GPU 연결 문제 | `--gpus all` 확인 후 CUDA 테스트 컨테이너 재실행 |
| `401 Client Error` 또는 `gated repo` | 모델 접근 권한 없음 | 접근 승인, Read 토큰 전달 또는 공개 모델 선택 |
| `port is already allocated` | 8000번 포트 사용 중 | 기존 컨테이너 종료 또는 `-p 8001:8000` 사용 |
| `cuda>=13.0` | 호스트 드라이버와 이미지 불일치 | CUDA 12 호환 이미지 태그 또는 승인된 드라이버 업데이트 |

문제 확인 순서:

```powershell
docker ps -a
docker logs --tail 100 컨테이너이름
```

실시간 로그:

```powershell
docker logs -f 컨테이너이름
```

## 12. 서버 실행 통과 기준

1. `http://localhost:8000/v1/models`에서 모델 이름을 확인합니다.
2. `/v1/chat/completions`로 한국어 응답을 한 번 받습니다.
3. Python OpenAI SDK에서도 같은 서버의 응답을 받습니다.
4. 내 GPU/RAM에서 어느 크기의 모델까지 실행할 수 있는지 설명할 수 있습니다.

## 13. 다음 주 수업과 과제

다음 주에는 다음 루프를 반복합니다.

```text
llmfit doctor
  → llmfit fit/info
  → Docker 또는 vLLM으로 모델 실행
  → llmfit bench
  → 속도나 품질이 부족하면 모델을 다시 선택
```

과제: **내 노트북 모델 카드 만들기**

1. `llmfit doctor` 결과 캡처
2. `llmfit fit` 화면 캡처
3. 추천 모델 중 하나를 실제로 실행하고 벤치마크 값 기록
4. 해당 모델을 선택한 이유와 사용 소감을 한 줄로 작성

다음 단계에서는 실행한 로컬 모델을 실제 웹 또는 챗봇 서비스와 연결할 예정입니다.

## 참고 자료

- [vLLM Docker 실행 문서](https://docs.vllm.ai/en/latest/deployment/docker/)
- [vLLM Metal](https://github.com/vllm-project/vllm-metal)
- [Scoop 설치](https://github.com/ScoopInstaller/Install)
- [NVIDIA CUDA 호환성](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/)
