# Changelog

이 포크(`mech12/microsoft-mcp`)의 변경 이력 및 트러블슈팅 기록.

원본: [elyxlz/microsoft-mcp](https://github.com/elyxlz/microsoft-mcp)

---

## 2026-03-26 — SharePoint 파일 다운로드 지원

### 변경 내용

**`src/microsoft_mcp/tools.py`**

- `search_files`: 결과에 `drive_id` (`parentReference.driveId`) 추가 반환
- `get_file`: `drive_id` 옵션 파라미터 추가
  - `drive_id` 있으면 `/drives/{driveId}/items/{itemId}` 경로 사용 (SharePoint)
  - `drive_id` 없으면 기존 `/me/drive/items/{itemId}` 경로 유지 (OneDrive)

### 배경

원본 `get_file`은 `/me/drive/items/{id}` (OneDrive 전용) 경로만 사용하여
SharePoint 파일 다운로드 시 404 Not Found 발생.
`search_files`는 SharePoint 파일도 검색 가능하지만(`/search/query` API 사용),
검색 결과의 `id`를 `get_file`에 전달하면 OneDrive에서 찾으려 하므로 실패함.

### 사용법

```python
# 1. search_files로 검색 → 결과의 drive_id 확인
results = search_files(query="동구바이오", account_id="...", limit=10)
# results[0]["drive_id"] = "b!BukLc3BkyEerLjFAgaheQw0qGYBDLI1Iu6rYr6WASedgYNmN78ZeR4KkZKqDqs_g"

# 2. get_file 호출 시 drive_id 전달
get_file(file_id=results[0]["id"], account_id="...", download_path="/tmp/file.md",
         drive_id=results[0]["drive_id"])
```

### 필요 권한

`Sites.Read.All` (Delegated) — Azure Portal에서 앱 등록 > API 권한에 추가 + Admin Consent 필요

---

## 2026-03-19 — Delegated scope 개별 지정 (Mail.Send 403 해결)

### 변경 내용

**`src/microsoft_mcp/auth.py`**

- `SCOPES = ["https://graph.microsoft.com/.default"]` 제거
- 개별 Delegated permission scope 목록(`DEFAULT_SCOPES`)으로 변경
- `MICROSOFT_MCP_SCOPES` 환경변수로 오버라이드 가능 (space-separated)

### 트러블슈팅: Mail.Send 403 Forbidden

#### 증상

- Azure Portal에서 `Mail.Send` 권한 추가 + Admin Consent 완료 (녹색 체크 확인)
- 재인증(Device Code Flow)을 여러 번 수행
- 그러나 `send_email` 호출 시 계속 **403 Forbidden** 발생

#### 원인

원본 코드의 scope가 `/.default`로 하드코딩되어 있었음:

```python
# 변경 전 (문제)
SCOPES = ["https://graph.microsoft.com/.default"]
```

- `/.default` scope는 **Application permissions (앱 권한)** 전용
- **Delegated permissions (위임된 권한)** 에서는 개별 scope를 명시해야 토큰에 해당 권한이 포함됨
- Azure Portal에서 권한을 추가해도, 토큰 요청 시 scope에 포함되지 않으면 Access Token에 `Mail.Send` 권한이 들어가지 않음

#### 해결

```python
# 변경 후 (해결)
DEFAULT_SCOPES = [
    "User.Read",
    "Mail.ReadWrite",
    "Mail.Send",
    "Calendars.ReadWrite",
    "Contacts.Read",
    "Files.ReadWrite",
    "People.Read",
    "Chat.ReadWrite",
    "ChannelMessage.Read.All",
    "ChannelMessage.Send",
    "Channel.ReadBasic.All",
    "Team.ReadBasic.All",
    "User.ReadBasic.All",
]

_env_scopes = os.getenv("MICROSOFT_MCP_SCOPES")
SCOPES = _env_scopes.split() if _env_scopes else DEFAULT_SCOPES
```

#### 시도한 방법 (시간순)

1. **Azure Portal 권한 추가 후 재인증** → 실패 (토큰 scope가 변경되지 않음)
2. **MCP 서버에 `MICROSOFT_MCP_SCOPES` 환경변수 추가** → 실패 (소스코드에서 환경변수를 읽지 않았음)
3. **소스코드 수정 + 토큰 캐시 삭제** → 성공

#### 적용 절차

```bash
# 1. 소스코드 수정 (auth.py의 SCOPES 변경)
# 2. 기존 토큰 캐시 삭제 (이전 scope로 발급된 토큰 제거)
rm ~/.microsoft_mcp_token_cache.json
# 3. Claude Code 재시작
# 4. Device Code Flow 재인증 (새 scope로 토큰 발급)
# 5. send_email 호출 → 성공
```

#### 핵심 교훈

| 항목 | 설명 |
|------|------|
| `/.default` scope | Application permissions 전용. Delegated permissions에서는 개별 scope 명시 필요 |
| Azure Portal 권한 추가 | 앱이 **요청할 수 있는** 권한을 등록하는 것. 실제 토큰에 포함되려면 scope로 요청해야 함 |
| Admin Consent | 앱이 해당 권한을 요청하는 것을 **조직이 허용**하는 것 |
| 토큰 캐시 | scope 변경 후 반드시 캐시 삭제 + 재인증 필요. 기존 토큰에는 이전 scope만 포함됨 |
