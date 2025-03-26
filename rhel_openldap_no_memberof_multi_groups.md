# 在 RHEL 5-9 中未啟用 `memberOf` 的 OpenLDAP 與 nscd/nslcd 多群組 UID 控管

## 概述
本文件說明在 OpenLDAP 未啟用 `memberOf` 覆蓋模組的情況下，如何在 RHEL 5/6/7/8/9 中使用 `nscd` 和 `nslcd`，基於 LDAP 的 `groupOfNames` 實現兩個以上群組的 UID 控管，並確保不影響本地帳號權限。目標包括：
- 配置多個 LDAP 群組（主群組與次群組）。
- 利用 GID 和檔案權限控管用戶存取。
- 保持本地帳號獨立性。

假設 OpenLDAP 基礎 DN 為 `dc=example,dc=com`，伺服器地址為 `ldap.example.com`。

---

## 前置條件
- **OpenLDAP 端**：
  - 已建立 `ou=people`（用戶）和 `ou=groups`（群組）。
  - 群組使用 `groupOfNames`，透過 `member` 屬性定義成員，未啟用 `memberOf`。
- **RHEL 端**：
  - 已安裝 `nss-pam-ldapd`、`openldap-clients`。

---

## OpenLDAP 結構範例
以下展示多群組結構：

### 用戶（`ou=people`）
```ldif
dn: uid=user1,ou=people,dc=example,dc=com
objectClass: posixAccount
objectClass: inetOrgPerson
uid: user1
cn: User One
uidNumber: 1001
gidNumber: 1001  # 主群組對應 cn=ldapusers
homeDirectory: /home/user1
loginShell: /bin/bash

dn: uid=user2,ou=people,dc=example,dc=com
objectClass: posixAccount
objectClass: inetOrgPerson
uid: user2
cn: User Two
uidNumber: 1002
gidNumber: 1001  # 主群組對應 cn=ldapusers
homeDirectory: /home/user2
loginShell: /bin/bash
多群組（ou=groups）
ldif

收起

換行

複製
# 主群組
dn: cn=ldapusers,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: ldapusers
gidNumber: 1001
member: uid=user1,ou=people,dc=example,dc=com
member: uid=user2,ou=people,dc=example,dc=com

# 次群組 1
dn: cn=developers,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: developers
gidNumber: 1002
member: uid=user1,ou=people,dc=example,dc=com

# 次群組 2
dn: cn=admins,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: admins
gidNumber: 1003
member: uid=user2,ou=people,dc=example,dc=com
說明：
user1 屬於 ldapusers（GID 1001，主群組）和 developers（GID 1002，次群組）。
user2 屬於 ldapusers（GID 1001，主群組）和 admins（GID 1003，次群組）。
RHEL 客戶端配置
1. /etc/openldap/ldap.conf
基礎配置，無 memberOf 相關設定。

範例：
text

收起

換行

複製
BASE dc=example,dc=com
URI ldap://ldap.example.com
TLS_CACERTDIR /etc/openldap/cacerts
2. /etc/nslcd.conf
配置 nslcd 映射多群組。

範例：
text

收起

換行

複製
uid nslcd
gid nslcd
uri ldap://ldap.example.com
base dc=example,dc=com
base passwd ou=people,dc=example,dc=com
base group ou=groups,dc=example,dc=com
filter passwd (objectClass=posixAccount)
filter group (objectClass=groupOfNames)
map passwd uid uid
map passwd uidNumber uidNumber
map passwd gidNumber gidNumber
map passwd homeDirectory homeDirectory
map passwd loginShell loginShell
map group gidNumber gidNumber
map group member member
關鍵：
filter group (objectClass=groupOfNames) 查詢所有群組。
map group member member 解析多群組的 member 屬性。
啟動：
bash

收起

換行

複製
systemctl enable nslcd
systemctl start nslcd
版本差異：
RHEL 5-7：預設支援。
RHEL 8/9：需安裝：
bash

收起

換行

複製
dnf install nss-pam-ldapd openldap-clients
3. /etc/nsswitch.conf
設定 NSS 查詢順序，支援多群組。

範例：
text

收起

換行

複製
passwd:     files ldap
group:      files ldap
shadow:     files ldap
效果：NSS 根據 member 屬性解析用戶的主群組和次群組。
4. /etc/pam.d/system-auth
確保本地帳號獨立，支援 LDAP 多群組用戶。

範例：
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
session     required      pam_unix.so
session     optional      pam_ldap.so
重點：
pam_succeed_if.so uid >= 1000 限制 LDAP 用戶範圍，保護本地帳號（如 root）。
版本差異：
RHEL 6/7：可用 authconfig：
bash

收起

換行

複製
authconfig --enableldap --enableldapauth --ldapserver=ldap.example.com --ldapbasedn=dc=example,dc=com --update
RHEL 8/9：禁用 SSSD：
bash

收起

換行

複製
systemctl disable sssd
5. /etc/nscd.conf
快取多群組查詢，提升效能。

範例：
text

收起

換行

複製
enable-cache passwd yes
positive-time-to-live passwd 3600
enable-cache group yes
positive-time-to-live group 3600
啟動：
bash

收起

換行

複製
systemctl enable nscd
systemctl start nscd
6. 可選：/etc/security/access.conf
限制登入，進一步保護本地帳號。

範例：
text

收起

換行

複製
+ : root : LOCAL
+ : @ldapusers : ALL
+ : @developers : ALL
+ : @admins : ALL
- : ALL : ALL
啟用：在 /etc/pam.d/sshd 加入：
text

收起

換行

複製
account    required   pam_access.so
多群組 UID 控管實例
驗證群組成員：
bash

收起

換行

複製
getent group ldapusers
# 輸出: ldapusers:1001:user1,user2
getent group developers
# 輸出: developers:1002:user1
getent group admins
# 輸出: admins:1003:user2
id user1
# 輸出: uid=1001(user1) gid=1001(ldapusers) groups=1001(ldapusers),1002(developers)
id user2
# 輸出: uid=1002(user2) gid=1001(ldapusers) groups=1001(ldapusers),1003(admins)
檔案權限控管：
建立目錄並分配權限：
bash

收起

換行

複製
mkdir -p /home/shared /home/dev /home/admin
chown :ldapusers /home/shared
chown :developers /home/dev
chown :admins /home/admin
chmod g+rw /home/shared /home/dev /home/admin
結果：
user1 可寫入 /home/shared 和 /home/dev。
user2 可寫入 /home/shared 和 /home/admin。
不影響本地帳號的保障
UID/GID 分隔：LDAP UID/GID >= 1000，本地帳號 < 1000。
NSS 優先級：files ldap 確保本地檔案優先。
PAM 限制：pam_succeed_if.so uid >= 1000 排除本地低 UID。
RHEL 版本差異
版本	nscd/nslcd 支援	配置工具	注意事項
RHEL 5	是	手動	基礎支援，無自動化工具
RHEL 6	是	authconfig	支援多群組解析
RHEL 7	是	authconfig	穩定支援
RHEL 8	是（需安裝）	手動	預設 SSSD，需禁用
RHEL 9	是（需安裝）	手動	同 RHEL 8
注意事項
效能：無 memberOf，多群組查詢依賴 member 屬性，nscd 快取可優化。
安全性：建議啟用 TLS（uri ldaps://）。
測試：修改後清除快取：
bash

收起

換行

複製
nscd -i passwd
nscd -i group
總結
在 OpenLDAP 未啟用 memberOf 的情況下，RHEL 5-9 可透過 nslcd 和 groupOfNames 的 member 屬性實現兩個以上 LDAP 群組的 UID 控管。用戶可同時隸屬多個群組（主群組 + 次群組），利用 GID 和檔案權限進行管理，並透過 NSS 和 PAM 配置確保本地帳號權限不受影響。

text

收起

換行

複製

---

### 保存與使用
1. 將內容複製到文字編輯器。
2. 保存為 `rhel_openldap_no_memberof_multi_groups.md`。
3. 上傳至 GitHub 儲存庫，與其他相關文件並存。

如果需要更具體的範例（例如更多群組或特定權限場景），請告訴我，我可以進一步調整！
