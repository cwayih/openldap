# RHEL 系統中 OpenLDAP 與 nscd/nslcd 的設定指南

## 概述
本文件說明如何在 RHEL 5/6/7/8/9 系列中，使用 `nscd`（Name Service Cache Daemon）和 `nslcd`（Name Service LDAP Daemon）搭配 OpenLDAP，提供身份驗證、群組管理和 `sudo` 權限。目標包括：
- 配置 OpenLDAP 客戶端以識別 `ou=people`、`ou=groups` 和 `sudoRole`。
- 啟用 `memberOf` 屬性並與 `sudo-ldap` 整合。
- 適應 RHEL 各版本的差異（RHEL 8/9 預設推 SSSD，但仍支援 nslcd）。

假設 OpenLDAP 基礎 DN 為 `dc=example,dc=com`，伺服器地址為 `ldap.example.com`。

---

## 前置條件
- **OpenLDAP 端**：已啟用 `memberOf`（見 `openldap_setup.md`），並設定：
  - `ou=people`（用戶）
  - `ou=groups`（群組，例如 `cn=ldapadmin,ou=groups`）
  - `ou=sudoers`（含 `sudoRole`，如 `cn=admins,ou=sudoers`）
- **RHEL 端**：已安裝 `nss-pam-ldapd`、`openldap-clients` 和 `sudo`。

---

## RHEL 客戶端配置

### 1. `/etc/openldap/ldap.conf`
OpenLDAP 客戶端通用配置檔案。

#### 範例：
BASE dc=example,dc=com
URI ldap://ldap.example.com
TLS_CACERTDIR /etc/openldap/cacerts
SUDOERS_BASE ou=sudoers,dc=example,dc=com

text

收起

換行

複製

#### 版本差異：
- **RHEL 5/6**：無 `SUDOERS_BASE`，需在 `sudo-ldap.conf` 中指定。
- **RHEL 7/8/9**：支援 `SUDOERS_BASE`，與 `sudo-ldap` 整合。

---

### 2. `/etc/nslcd.conf`
`nslcd` 負責與 LDAP 伺服器通訊，需安裝 `nss-pam-ldapd`。

#### 範例：
uid nslcd
gid nslcd
uri ldap://ldap.example.com
base dc=example,dc=com
base group ou=groups,dc=example,dc=com
base passwd ou=people,dc=example,dc=com
filter passwd (objectClass=posixAccount)
filter group (objectClass=groupOfNames)
map passwd uid uid
map passwd gidNumber gidNumber
map passwd homeDirectory homeDirectory
map passwd loginShell loginShell
map group member member

text

收起

換行

複製

#### 版本差異：
- **RHEL 5/6**：基本一致，無顯著變化。
- **RHEL 7**：支援更多映射選項。
- **RHEL 8/9**：需手動安裝 `nss-pam-ldapd`（預設推 SSSD）：
  ```bash
  dnf install nss-pam-ldapd openldap-clients
啟動服務：
bash

收起

換行

複製
systemctl enable nslcd
systemctl start nslcd
3. /etc/sudo-ldap.conf
配置 sudo 從 LDAP 讀取權限，需安裝 sudo 並啟用 LDAP 支援。

範例：
text

收起

換行

複製
URI ldap://ldap.example.com
BASE dc=example,dc=com
SUDOERS_BASE ou=sudoers,dc=example,dc=com
BINDDN cn=admin,dc=example,dc=com
BINDPW admin_password
版本差異：
RHEL 5：無此獨立檔案，需調整 /etc/ldap.conf 或本地 /etc/sudoers。
RHEL 6/7/8/9：支援獨立檔案，與 sudoRole 整合。
配置步驟：
安裝 sudo：
bash

收起

換行

複製
yum install sudo  # RHEL 5-7
dnf install sudo  # RHEL 8-9
編輯 /etc/nsswitch.conf：
text

收起

換行

複製
sudoers: files ldap
4. /etc/pam.d 設定
調整 system-auth 和 sshd，讓 LDAP 用戶可登入並使用 sudo。

範例（/etc/pam.d/system-auth）：
text

收起

換行

複製
auth        required      pam_env.so
auth        sufficient    pam_unix.so nullok try_first_pass
auth        sufficient    pam_ldap.so use_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet
auth        required      pam_deny.so

account     required      pam_unix.so
account     sufficient    pam_ldap.so
account     required      pam_deny.so

password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    sufficient    pam_ldap.so use_authtok
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
session     optional      pam_ldap.so
範例（/etc/pam.d/sshd）：
text

收起

換行

複製
auth       required   pam_sepermit.so
auth       sufficient pam_ldap.so
auth       sufficient pam_unix.so nullok try_first_pass
auth       required   pam_deny.so

account    required   pam_unix.so
account    sufficient pam_ldap.so
account    required   pam_deny.so
版本差異：
RHEL 5：手動配置 pam_ldap.so。
RHEL 6/7：可用 authconfig：
bash

收起

換行

複製
authconfig --enableldap --enableldapauth --update
RHEL 8/9：若用 nslcd，需禁用 SSSD：
bash

收起

換行

複製
systemctl disable sssd
5. /etc/nsswitch.conf
確保 NSS 使用 LDAP。

範例：
text

收起

換行

複製
passwd:     files ldap
group:      files ldap
shadow:     files ldap
sudoers:    files ldap
版本差異：
RHEL 5-7：直接支援。
RHEL 8/9：若用 nslcd，需明確指定 ldap。
6. /etc/nscd.conf
nscd 快取 NSS 查詢，提升效能。

範例：
text

收起

換行

複製
enable-cache passwd yes
positive-time-to-live passwd 3600
enable-cache group yes
positive-time-to-live group 3600
啟動服務：
bash

收起

換行

複製
systemctl enable nscd
systemctl start nscd
版本差異：
RHEL 5-9：一致支援。
活用 LDAP group 和 memberOf
假設 OpenLDAP 已設定 cn=ldapadmin,ou=groups 和 cn=admins,ou=sudoers（sudoUser: %ldapadmin）：

群組與 sudo 關聯：
用戶加入 cn=ldapadmin,ou=groups，其 memberOf 屬性自動更新。
sudo-ldap 根據 sudoRole 賦予權限。
驗證：
bash

收起

換行

複製
getent group ldapadmin
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "dc=example,dc=com" "(memberOf=cn=ldapadmin,ou=groups,dc=example,dc=com)"
sudo -l -U <username>
RHEL 版本差異總結
版本	nscd/nslcd 支援	sudo-ldap 支援	預設工具	配置工具
RHEL 5	是	有限（需本地調整）	nslcd	手動
RHEL 6	是	是	nslcd	authconfig
RHEL 7	是	是	nslcd	authconfig
RHEL 8	是（需安裝）	是	SSSD	手動/SSSD
RHEL 9	是（需安裝）	是	SSSD	手動/SSSD
注意事項
安全性：建議啟用 TLS（修改 URI 為 ldaps:// 並配置證書）。
測試：每次修改後，重啟 nslcd 和 nscd：
bash

收起

換行

複製
systemctl restart nslcd nscd
SSSD 替代：RHEL 8/9 若不想用 nslcd，可考慮 SSSD，配置更簡化但需調整策略。
總結
本指南提供 RHEL 5-9 中使用 nscd 和 nslcd 搭配 OpenLDAP 的完整設定，實現身份驗證和 sudo 權限管理，並活用 memberOf 屬性進行群組授權。

text

收起

換行

複製

---

### 保存與使用
1. 將上述內容複製到文字編輯器。
2. 保存為 `rhel_openldap_nscd_nslcd_setup.md`。
3. 上傳至你的 GitHub 儲存庫，與 `openldap_setup.md` 並存。

如果需要針對特定版本深入說明，或添加範例用戶測試流程，請告訴我！
