# Supabase 원본

- 소스: https://github.com/supabase/supabase/tree/self-hosted/v0.8.1/docker
- 릴리스: `self-hosted/v0.8.1`
- 원본 라이선스: `LICENSE.upstream`

공식 `docker-compose.yml`에 동일 릴리스의 `docker-compose.kong.yml`을 반영했다.
기획서 Ch.4의 Kong 구성을 유지하며, 필요한 `volumes/` 파일을 함께 포함했다.
각 서비스 이미지는 해당 릴리스의 고정 태그를 유지한다.

수업용 변경:

- Compose 프로젝트 이름 `notemind-supabase`, 파일명 `compose.yml`.
- API/Studio `127.0.0.1:54321`, DB 풀러 `127.0.0.1:54322/54329`.
- 사용하지 않는 HTTPS 포트 매핑 제거.
- `.env` 필수 시크릿 누락 시 Compose 실패. `../init-env.py`로 생성.
- 공식 스택이 지원하는 HS256 JWT / anon / service_role 키 사용.
  비대칭 JWT 및 새 opaque API 키는 활성화하지 않았다.
- 외부 SMTP 없이 이메일/비밀번호 실습을 위해 이메일 자동 확인 사용.
- 전화 가입 비활성화, Edge Functions JWT 검증 활성화.
- `vector.sql`로 pgvector 활성화. 앱 테이블과 RLS는 `migrations/001_notemind.sql`로 적용.
- 클라우드 OpenAI 키는 빈 값. 공식 스택의 선택적 Analytics는 포함하지 않음.

원본의 선택적 override 파일과 관리 스크립트는 포함하지 않았다.
관리는 이 폴더에서 `docker compose` 명령으로 수행한다.

실기동 수정: DB 및 Storage를 이름 있는 Docker 볼륨으로 변경했다. Mac 파일 공유의
소유권 오류와 Storage 확장 속성 오류를 방지한다. DB healthcheck도 실제 SQL 실행으로 확인한다.
