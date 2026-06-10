# useradd.d 스크립트 멱등성 분석

## 개요
- /etc/gumd/useradd.d의 스크립트가 여러번 반복해서 수행되어도 동일한 결과를 도출하는지(멱등성) 조사
- MIC단계에서 한번 수행되고, 첫부팅 시 platform-extension migration script에 의해 다시한번 실행될 경우 부작용 없는지 검토

## 분석 대상

| 스크립트 | 실행 명령 |
|---|---|
| `10_package-manager-add.post` | `pkg_initdb --uid $2` 외 |
| `11_notification-add.post` | `notification_init --uid $2` |
| `12_appsvc-add.post` | sqlite3로 appsvc DB 생성 |
| `14_component-add.post` | sqlite3로 component DB 생성 |
| `30_media-controller-add.post` | touch로 media controller DB 파일 생성 |
| `30_media-server-add.post` | touch로 media DB 및 thumb 디렉토리 생성 |
| `50_security-manager-add.post` | `security-manager-cmd --manage-users=add` |
| `90_user-content-permissions.post` | find로 콘텐츠 디렉토리 권한 설정 |
| `91_user-dbspace-permissions.post` | chown/chmod로 사용자 홈 권한 설정 |

---

## 1. 스크립트 자체 멱등성 분석

### 안전 (Idempotent)

#### `12_appsvc-add.post`
- `mkdir -p`: 디렉토리 존재 여부에 무관하게 안전
- SQL: `CREATE TABLE IF NOT EXISTS`, `CREATE TRIGGER IF NOT EXISTS` 사용으로 중복 실행 안전
- **주의**: `.appsvc.db-journal` 파일이 첫 실행 시 존재하지 않을 수 있어 해당 파일의 `chmod`/`chown`/`chsmack` 명령이 조용히 실패함 (스크립트에 `-e` 플래그 없음)

#### `14_component-add.post`
- `12_appsvc-add.post`와 동일한 구조로 멱등함
- **주의**: `.component.db-journal` 파일의 동일한 문제 존재
- **버그**: 19번 라인 `PAAGMA user_version = 1;` → `PRAGMA` 오타로 해당 SQL이 조용히 실패함

#### `30_media-controller-add.post` / `30_media-server-add.post`
- `touch`: 파일 생성 또는 mtime 갱신, 존재 여부 무관하게 안전
- `chown`, `chmod`, `chsmack`, `mkdir -p`: 모두 멱등함

#### `90_user-content-permissions.post`
- `find + chown + chmod`: 반복 실행해도 동일 결과

#### `91_user-dbspace-permissions.post`
- `chown`, `chmod`: 멱등함
- `if [ ! -d $TZ_USER_DB/privacy ]` 조건 후 `mkdir`: 중복 생성 방지

---

### 소스코드 분석 기반 판정

#### `11_notification-add.post` → `notification_init --uid $2`

**판정: 멱등함**

소스(`notification_setting.c`)를 분석한 결과, 두 내부 함수 모두 INSERT 전 존재 확인 후 조건부 삽입 방식으로 구현되어 있음.

| 내부 함수 | 멱등성 메커니즘 |
|---|---|
| `notification_setting_refresh_setting_table()` | `_is_package_in_setting_table()` 확인 → 존재 시 SKIP |
| `notification_system_setting_init_system_setting_table()` | `_is_uid_in_system_setting_table()` 확인 → 존재 시 즉시 반환 |

DB 스키마에도 `UNIQUE(uid, package_name, app_id)` 제약이 있어 코드 레벨과 DB 레벨 이중 보호됨.

#### `50_security-manager-add.post` → `security-manager-cmd --manage-users=add`

**판정: 멱등함 (동일 인자 재실행 기준)**

| 구성요소 | 멱등성 메커니즘 | 근거 |
|---|---|---|
| Cynara 정책 | `cynara_admin_set_policies()` 덮어쓰기 방식 | `cynara.cpp:485-520` |
| Permissible 파일 | `mkdir()` EEXIST 무시, `ofstream` truncate 재생성 | `permissible-set.cpp:290-304` |

**주의**: `--usertype`이 변경된 채로 재실행하면 Cynara에 새 타입 정책이 추가되지만 이전 타입 정책이 정리되지 않음.

#### `10_package-manager-add.post` → `pkg_initdb --uid $2`

**판정: 멱등함 (단, 데이터 파괴적 방식)**

`HandleDatabase()`(`init_pkg_db.cc:131-142`) 구현:

```
RemoveOldDatabases(uid)  →  기존 DB 파일 삭제 (없으면 무시)
pkgmgr_parser_create_and_initialize_db(uid)  →  빈 DB 새로 생성
```

`Remove()` 함수(`file_util.cc:365-375`)는 파일이 없어도 `true`를 반환하므로 재실행 시 오류 없음.
매 실행 시 최종 상태가 동일하여 멱등성은 보장되나, **재실행 시 해당 UID의 기존 패키지 DB 데이터가 초기화됨**.

---

## 2. Tizen Read-Only 영역 관점 분석

### Tizen 런타임 파일시스템 구조

```
/              (rootfs, Read-Only)  →  /usr, /lib, /bin, /sbin, /etc
/opt           (data partition, Read-Write)
/opt/dbspace   =  TZ_SYS_DB
/opt/var       =  TZ_SYS_VAR
/tmp           (tmpfs, Read-Write)
```

### `pkg_initdb` 경로 분석

| 동작 | 접근 경로 | 접근 유형 | 런타임 안전성 |
|---|---|---|---|
| DB 재생성 | `TZ_SYS_DB/user/{uid}/` (`/opt/dbspace/user/{uid}/`) | Write | 안전 ✓ |
| 글로벌 패키지 읽기 | `TZ_SYS_RO_PACKAGES` (`/usr/share/packages` 등) | **Read-only** | 읽기만, 안전 ✓ |
| 사용자 패키지 처리 | `TZ_USER_PACKAGES` (`/opt/` 하위) | Read/Write | 안전 ✓ |

`TZ_SYS_RO_PACKAGES`는 읽기 전용 접근만 수행하며, `--uid $2`로 특정 사용자를 지정할 경우 해당 경로에는 아예 접근하지 않음 (`IsGlobal() == false`).

### `notification_init` 경로 분석

```c
// notification_db.h
#define DBPATH tzplatform_mkpath(TZ_SYS_DB, ".notification.db")
// → /opt/dbspace/.notification.db
```

전역 notification DB는 `/opt/dbspace/.notification.db`로 런타임에도 쓰기 가능한 영역.

**잠재적 문제**: `notification_db_open()`은 `access(DBPATH, R_OK | W_OK)` 로 파일 존재를 확인한 뒤 열기를 시도함. 전역 DB가 존재하지 않으면 에러 없이 NULL 반환 후 조용히 실패. 스크립트 자체에도 shebang과 에러 처리가 없어 실패가 전파되지 않음.

### `security-manager-cmd` 경로 분석

`security-manager.spec` line 175:
```
-DLOCAL_STATE_INSTALL_PREFIX=%{TZ_SYS_VAR}
```

CMake 기본값인 `/var` (rootfs, Read-Only) 대신 Tizen 전용 변수 `%{TZ_SYS_VAR}`(`/opt/var`)를 **빌드 옵션으로 명시적 재지정**하여 Read-Only 영역 쓰기 문제를 회피함.

| 동작 | 접근 경로 | 런타임 안전성 |
|---|---|---|
| Permissible 파일 생성 | `/opt/var/security-manager/{uid}/apps-labels` | 안전 ✓ |
| Cynara 정책 (daemon 실행 중) | IPC → daemon이 처리 | 안전 ✓ |
| Cynara 정책 (offline mode) | `/opt/var/` 하위 Cynara DB | 안전 ✓ |

---

## 3. 최종 종합 판정표

| 스크립트 | 멱등성 | Read-Only 문제 | 비고 |
|---|---|---|---|
| `10_package-manager-add.post` | **멱등함** | 없음 | 기존 DB 삭제 후 재생성 방식, 데이터 초기화 주의 |
| `11_notification-add.post` | **멱등함** | 없음 | 전역 DB 미존재 시 조용히 실패 |
| `12_appsvc-add.post` | **멱등함** | 없음 | journal 파일 미존재 시 chmod 등 조용히 실패 |
| `14_component-add.post` | **멱등함** | 없음 | journal 파일 문제 + `PAAGMA` 오타 버그 |
| `30_media-controller-add.post` | **멱등함** | 없음 | |
| `30_media-server-add.post` | **멱등함** | 없음 | |
| `50_security-manager-add.post` | **멱등함** | 없음 | `TZ_SYS_VAR`로 `/opt/var` 사용, usertype 변경 재실행 주의 |
| `90_user-content-permissions.post` | **멱등함** | 없음 | |
| `91_user-dbspace-permissions.post` | **멱등함** | 없음 | |

---

## 4. 잔여 개선 사항

Read-Only 영역 문제와 무관하게 개선이 필요한 항목:

1. **`14_component-add.post` 오타 수정**
   - 19번 라인: `PAAGMA user_version = 1;` → `PRAGMA user_version = 1;`

2. **`11_notification-add.post` 실패 감지 불가**
   - 스크립트가 단순 명령어 한 줄로만 구성되어 있어, `notification_init` 내부 실패가 외부로 전파되지 않음
   - shebang 및 종료 코드 확인 로직 추가 검토 필요

3. **`12_appsvc-add.post`, `14_component-add.post`의 journal 파일 처리**
   - 첫 실행 시 `.db-journal` 파일이 존재하지 않아 `chmod`/`chown`/`chsmack` 명령이 실패
   - `-e` 플래그가 없어 조용히 무시되지만, 실패 조건에 대한 처리 필요
