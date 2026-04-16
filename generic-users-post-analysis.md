# generic-users.post 동작 분석

## 개요

`generic-users.post` → `gum-utils --offline --add-user` → `gumd_daemon_user_add()` → 홈 디렉토리(`/opt/usr/home/owner`) 생성 → `useradd.d` 스크립트 실행

---

## 동작 순서

| Step | 건드리는 파일 | 설명 | 코드 및 파일 위치 |
|------|--------------|------|------------------|
| **1** owner 존재 확인 | `/etc/passwd`, `/opt/etc/passwd` (getent 경유) | `getent passwd`에서 `owner:` 항목 유무 확인. 이미 존재하면 이후 동작 전부 스킵 | `meta-generic/scripts/generic-users.post`<br>`generic_base_user_exists owner`<br>→ `getent passwd \| grep -q ^owner:`<br>(함수 정의: `generic-base.post:37`) |
| **2** gumd.conf 백업 | `/etc/gumd/gumd.conf` → `/etc/gumd/gumd.conf.orig` | 원본 conf를 `.orig`로 이동하여 임시 수정 전 백업 | `meta-generic/scripts/generic-users.post`<br>`conf=/etc/gumd/gumd.conf`<br>`origf=${conf}.orig`<br>`mv -v $conf $origf` |
| **3** gumd.conf 임시 수정 | `/etc/gumd/gumd.conf` (새로 생성) | `PASSWD_FILE` · `SHADOW_FILE` 경로를 `/opt/etc/` → `/etc/` 로 변경. owner는 Tizen 기본 유저이므로 항상 접근 가능한 `/etc/passwd`에 등록해야 함 | `meta-generic/scripts/generic-users.post`<br>`sed -e 's,^\(PASSWD_FILE\).*,\1=/etc/passwd,'`<br>`    -e 's,^\(SHADOW_FILE\).*,\1=/etc/shadow,'`<br>`    <$origf >$conf`<br>원본 값: `PASSWD_FILE=/opt/etc/passwd` (`rootfs/etc/gumd/gumd.conf:29`) |
| **4** gum-utils 호출 | (이후 4-x 스텝 참조) | `--offline` 모드로 DBus 없이 libgum 동기 API 직접 호출하여 owner 유저 추가 | `meta-generic/scripts/generic-users.post`<br>`gum-utils --offline --add-user --username=owner --usertype=admin --usecret=""`<br>→ `_handle_user_add()` (`gumd/src/utils/gumd-utils.c:267`)<br>→ `gum_user_create_sync(offline=TRUE)`<br>→ `gum_user_add_sync()` |
| **4-1** 유저타입·셸 검증 | (메모리 내 처리) | usertype=`admin` 확인. shell 미지정 시 config의 기본값(`/bin/bash`) 적용 | `gumd/src/daemon/core/gumd-daemon-user.c:1733`<br>`if (usertype == NONE) → error`<br>`if (!pw_shell) _set_shell_property(config→SHELL)` |
| **4-2** passwd·shadow 파일 존재 확인 | `/etc/passwd`, `/etc/shadow` | 파일이 없으면 빈 파일 생성 | `gumd/src/daemon/core/gumd-daemon-user.c:1743`<br>`if (!g_file_test(passwd_file, EXISTS))`<br>`    gum_file_create_db_files(passwd_file, ...)` |
| **4-3** UID 자동 할당 | `/etc/passwd` (읽기)<br>임시수정된 `/etc/passwd` (읽기) | `/etc/passwd`와 `PASSWD_FILE` 양쪽 모두 검사해 중복 없는 첫 빈 UID 선택. `UID_MIN+1=5001` 부터 순차 탐색 | `gumd/src/daemon/core/gumd-daemon-user.c:804` `_find_free_uid()`<br>`gum_file_getpwuid(tmp_uid, "/etc/passwd")`<br>`gum_file_getpwuid(tmp_uid, config→PASSWD_FILE)`<br>범위: `UID_MIN=5000`, `UID_MAX=9999` (`rootfs/etc/gumd/gumd.conf:83,87`) |
| **4-4** 홈 디렉토리 경로 결정 | (메모리 내 처리) | `HOME_DIR_PREF` + `/owner` 로 경로 결정. `HOME_DIR_PREF`는 `tzplatform_getenv(TZ_SYS_HOME)` 반환값. Tizen 환경에서 `TZ_SYS_HOME=/opt/usr/home` → **`/opt/usr/home/owner`** | `gumd/src/daemon/core/gumd-daemon-user.c:855`<br>`dir = g_strdup_printf("%s/%s", HOME_DIR_PREF, pw_name)`<br>`gumd/src/common/gum-config.c:625`<br>`home_path = tzplatform_getenv(TZ_SYS_HOME)` |
| **4-5** primary 그룹 설정 | `/etc/group` (읽기·쓰기) | `USR_PRIMARY_GRPNAME=users` 설정에 따라 `/etc/group`에서 `users` 그룹 탐색. 존재하면 해당 GID 사용, 없으면 새 그룹 생성 | `gumd/src/daemon/core/gumd-daemon-user.c:1162` `_set_group()`<br>`gum_file_getgrnam("users", config→GROUP_FILE)`<br>GROUP_FILE: `rootfs/etc/gumd/gumd.conf:43` → `/etc/group` |
| **4-6** shadow 데이터 설정 | (메모리 내 처리) | `--usecret=""` 이므로 빈 패스워드 → shadow의 `sp_pwdp`에 `""` 저장. 패스워드 정책(min/max 날짜 등) 초기화 | `gumd/src/daemon/core/gumd-daemon-user.c:902` `_set_shadow_data()`<br>`if (strcmp(pw_passwd, "") == 0)`<br>`    shadow->sp_pwdp = g_strdup("")` |
| **4-7** /etc/passwd 에 항목 추가 | `/etc/passwd` (쓰기) | UID 순서로 정렬 삽입<br>형식: `owner:x:5001:<gid>:owner,,,admin:/opt/usr/home/owner:/bin/bash` | `gumd/src/daemon/core/gumd-daemon-user.c:1769`<br>`gum_file_update(obj, GUM_OPTYPE_ADD,`<br>`    _update_passwd_entry, passwd_file, ...)` |
| **4-8** /etc/shadow 에 항목 추가 | `/etc/shadow` (쓰기) | shadow 항목 추가. sp_pwdp = "" (빈 패스워드) | `gumd/src/daemon/core/gumd-daemon-user.c:1772`<br>`gum_file_update(obj, GUM_OPTYPE_ADD,`<br>`    _update_shadow_entry, shadow_file, ...)` |
| **4-9** userinfo 파일 생성 | `/var/lib/gumd/user/<uid>` (쓰기) | icon 등 추가 유저 정보를 키파일 형식으로 저장 | `gumd/src/daemon/core/gumd-daemon-user.c:1779`<br>`_add_userinfo(self)`<br>`g_key_file_save_to_file(key, path, ...)`<br>경로: `USERINFO_DIR=/var/lib/gumd/user/` (`rootfs/etc/gumd/gumd.conf:69`) |
| **4-10** 기본 그룹 멤버십 추가 | `/etc/group` (읽기) | `DEFAULT_ADMIN_GROUPS` 확인. gumd.conf에 설정 없으므로 추가 그룹 없음 | `gumd/src/daemon/core/gumd-daemon-user.c:1781`<br>`_set_default_groups(self, error)`<br>참조: `DEFAULT_ADMIN_GROUPS` 미설정 (`rootfs/etc/gumd/gumd.conf:18`) |
| **4-11** 홈 디렉토리 생성 | `/opt/usr/home/owner` (생성)<br>`/opt/etc/skel` (읽기, 복사 원본) | `/opt/usr/home/owner` 디렉토리 생성 후 `SKEL_DIR` 내용 재귀 복사. `lchown`으로 소유권 설정, SMACK64 레이블 `User::Home` 적용 | `gumd/src/daemon/core/gumd-daemon-user.c:1783`<br>`_create_home_dir()`<br>→ `gum_file_create_home_dir(pw_dir, uid, gid, umask, ...)`<br>(`gumd/src/common/gum-file.c:847`)<br>`g_mkdir_with_parents(home_dir, mode)`<br>`_copy_dir_recursively(skel_dir, home_dir, uid, gid, ...)`<br>`lchown(home_dir, uid, gid)`<br>skel 원본: `SKEL_DIR=/opt/etc/skel` (`rootfs/etc/gumd/gumd.conf:66`) |
| **4-12** useradd.d 스크립트 이름순 실행 | `/etc/gumd/useradd.d/` 하위 스크립트 | 디렉토리 내 스크립트를 파일명 알파벳순으로 실행. 각 스크립트에 `owner 5001 <gid> /opt/usr/home/owner admin` 인수 전달 | `gumd/src/daemon/core/gumd-daemon-user.c:1792`<br>`scrip_dir = USERADD_SCRIPT_DIR`<br>(`gumd/src/daemon/core/Makefile.am:8`)<br>→ `"${sysconfdir}/gumd/useradd.d"` = `/etc/gumd/useradd.d`<br>`gum_utils_run_user_scripts(scrip_dir, "owner", uid, gid, homedir, "admin")`<br>(`gumd/src/common/gum-utils.c:296`) |
| **4-12a** `10_package-manager-add.post` | `/opt/dbspace/user/` (생성)<br>`TZ_USER_APP` 하위 디렉토리 | `/opt/dbspace/user/$uid` 디렉토리 생성, 앱 디렉토리 권한 설정, `pkg_initdb` 실행 | `rootfs/etc/gumd/useradd.d/10_package-manager-add.post`<br>`mkdir -p -Z User::Home -m 770 /opt/dbspace/user/$2`<br>`chown $TZ_USER_NAME:users /opt/dbspace/user/$2`<br>`pkg_initdb --uid $2` |
| **4-12b** `11_notification-add.post`<br>`12_appsvc-add.post`<br>`14_component-add.post` | 각 서비스별 DB·디렉토리 | 알림·앱서비스·컴포넌트 서비스의 유저별 DB 및 디렉토리 초기화 | `rootfs/etc/gumd/useradd.d/11_notification-add.post`<br>`rootfs/etc/gumd/useradd.d/12_appsvc-add.post`<br>`rootfs/etc/gumd/useradd.d/14_component-add.post` |
| **4-12c** `30_media-server-add.post`<br>`30_media-controller-add.post` | 미디어 서비스 관련 DB·디렉토리 | 미디어 서버·컨트롤러 관련 유저별 디렉토리 및 DB 초기화 | `rootfs/etc/gumd/useradd.d/30_media-server-add.post`<br>`rootfs/etc/gumd/useradd.d/30_media-controller-add.post` |
| **4-12d** `50_security-manager-add.post` | (security-manager 내부) | 보안 매니저에 신규 유저를 `admin` 타입으로 등록 | `rootfs/etc/gumd/useradd.d/50_security-manager-add.post`<br>`security-manager-cmd --manage-users=add --uid=$2 --usertype=$5` |
| **4-12e** `90_user-content-permissions.post` | `TZ_USER_CONTENT` 하위 디렉토리 | 미디어 콘텐츠 디렉토리 그룹 소유권을 `priv_mediastorage`로 설정, 권한 조정 | `rootfs/etc/gumd/useradd.d/90_user-content-permissions.post`<br>`find $TZ_USER_CONTENT -type d`<br>`    -exec chown root:priv_mediastorage {} +`<br>`    -exec chmod 2777 {} +` |
| **4-12f** `91_user-dbspace-permissions.post` | `/opt/dbspace/user/` | dbspace 디렉토리 권한 최종 정리 | `rootfs/etc/gumd/useradd.d/91_user-dbspace-permissions.post` |
| **5** gumd.conf 원복 | `/etc/gumd/gumd.conf` (복구) | 임시 수정했던 conf를 원래 파일로 복구 (`PASSWD_FILE` 다시 `/opt/etc/passwd`로 복원) | `meta-generic/scripts/generic-users.post`<br>`mv -v $origf $conf` |

---

## /opt/usr/home/owner 생성 경위

```
[generic-users.post]
  gumd.conf 수정: PASSWD_FILE=/etc/passwd, SHADOW_FILE=/etc/shadow
          ↓
  gum-utils --offline --add-user --username=owner --usertype=admin --usecret=""
          ↓
[gumd-daemon-user.c: gumd_daemon_user_add()]
  _set_uid()
    ├─ /etc/passwd      ← 중복 uid 체크 (하드코딩)
    ├─ /etc/passwd      ← 중복 uid 체크 (수정된 conf 기준, 동일)
    └─ HOME_DIR_PREF = tzplatform_getenv(TZ_SYS_HOME) = "/opt/usr/home"
       homedir = "/opt/usr/home" + "/" + "owner" = "/opt/usr/home/owner"
          ↓
  _create_home_dir()
    ├─ g_mkdir_with_parents("/opt/usr/home/owner", mode)
    ├─ skel_dir = SKEL_DIR = "/opt/etc/skel"  ← 복사 원본
    ├─ _copy_dir_recursively("/opt/etc/skel" → "/opt/usr/home/owner")
    ├─ lchown("/opt/usr/home/owner", uid=5001, gid)
    └─ setxattr → SMACK64 "User::Home"
```

### HOME_DIR_PREF 결정 흐름

`gumd.conf`에서 `HOME_DIR` 항목이 주석 처리(`#HOME_DIR=/home`)되어 있으므로,
코드 기본값인 `tzplatform_getenv(TZ_SYS_HOME)` 이 사용됩니다.

```
gum-config.c:625
  home_path = tzplatform_getenv(TZ_SYS_HOME);
  if (home_path != NULL)
      HOME_DIR_PREF = home_path;   // "/opt/usr/home"  (Tizen 환경)
  else
      HOME_DIR_PREF = "/home";     // fallback
```

---

## 참조 파일 목록

| 파일 경로 | 역할 |
|-----------|------|
| `meta-generic/scripts/generic-users.post` | 진입점 스크립트 |
| `meta-generic/scripts/generic-base.post` | `generic_base_user_exists` 함수 정의 |
| `rootfs/etc/gumd/gumd.conf` | gumd 설정 파일 (PASSWD_FILE, SKEL_DIR, UID_MIN 등) |
| `gumd/src/utils/gumd-utils.c` | `gum-utils` 명령어 메인 (옵션 파싱, `_handle_user_add`) |
| `gumd/src/daemon/core/gumd-daemon-user.c` | 유저 추가 핵심 로직 (`gumd_daemon_user_add`) |
| `gumd/src/common/gum-config.c` | 설정 로드, `HOME_DIR_PREF` 결정 (`tzplatform_getenv`) |
| `gumd/src/common/gum-file.c` | 홈 디렉토리 생성·skel 복사 (`gum_file_create_home_dir`) |
| `gumd/src/common/gum-utils.c` | useradd.d 스크립트 실행 (`gum_utils_run_user_scripts`) |
| `gumd/src/daemon/core/Makefile.am` | `USERADD_SCRIPT_DIR` 컴파일 타임 정의 |
| `/etc/passwd`, `/etc/shadow` | owner 항목이 기록되는 시스템 DB (임시 수정된 conf 기준) |
| `/etc/group` | primary 그룹(`users`) 탐색·생성 |
| `/opt/etc/skel` | 홈 디렉토리 복사 원본 (skeleton) |
| `/opt/usr/home/owner` | **최종 생성되는 홈 디렉토리** |
| `/var/lib/gumd/user/<uid>` | userinfo 키파일 (icon 등) |
| `rootfs/etc/gumd/useradd.d/` | 유저 추가 후 실행되는 훅 스크립트 디렉토리 |
