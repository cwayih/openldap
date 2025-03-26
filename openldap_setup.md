我會將剛才的 `openldap_setup.md` 內容進行補遺，加入你提到的 `sudoRole` 結構設定，並保持原有的結構完整。以下是更新後的 `openldap_setup.md` 內容：

```markdown
# OpenLDAP 設定問題與解答

## 提問
如果在空的 OpenLDAP server 容器要建立 `ou=people`、`ou=groups`，開啟 `memberof`，`cn=ldapadmin,ou=groups` 讓這個 `ldapadmin` 等效於 `cn=admin`，這樣的 LDIF 如何設定及執行？

## 回答

### 概述
以下說明如何在空的 OpenLDAP 伺服器容器中設定：
- 建立 `ou=people` 和 `ou=groups`。
- 啟用 `memberOf` 覆蓋模組。
- 建立 `cn=ldapadmin,ou=groups` 並透過 ACL 賦予其與 `cn=admin` 等效的管理權限。
- **補遺**：設定 `sudoRole` 結構，與 `cn=ldapadmin,ou=groups` 整合以支援 `sudo` 權限。

假設基礎 DN 是 `dc=example,dc=com`，請根據實際情況替換。

---

### 步驟 1：啟用 `memberOf` 覆蓋模組

#### LDIF 文件：`enable_memberof.ldif`
```ldif
# 載入 memberof 模組
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof

# 為資料庫啟用 memberof 覆蓋模組
dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: olcConfig
objectClass: top
olcOverlay: memberof
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAD: memberOf
```

#### 執行命令
```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f enable_memberof.ldif
```

**說明**：
- `-Y EXTERNAL` 使用外部認證（root 權限）。
- `-H ldapi:///` 使用本地 Unix socket。
- 若後端不是 `mdb`，需調整 `olcDatabase={1}mdb`。

---

### 步驟 2：建立基本結構 (`ou=people` 和 `ou=groups`)

#### LDIF 文件：`base_structure.ldif`
```ldif
# 建立 ou=people
dn: ou=people,dc=example,dc=com
objectClass: organizationalUnit
ou: people

# 建立 ou=groups
dn: ou=groups,dc=example,dc=com
objectClass: organizationalUnit
ou: groups
```

#### 執行命令
```bash
ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f base_structure.ldif
```

**說明**：
- `-x` 使用簡單認證。
- `-D` 指定管理員 DN。
- `-W` 提示輸入密碼（預設可能是 `admin`）。

---

### 步驟 3：建立 `cn=ldapadmin,ou=groups` 並使其等效於 `cn=admin`

#### LDIF 文件：`create_ldapadmin_group.ldif`
```ldif
# 建立 ldapadmin 群組
dn: cn=ldapadmin,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: ldapadmin
description: LDAP Administrators Group
member: cn=admin,dc=example,dc=com
```

#### 執行命令
```bash
ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f create_ldapadmin_group.ldif
```

---

### 步驟 4：設定 ACL 讓 `cn=ldapadmin,ou=groups` 有管理權限

#### LDIF 文件：`set_acl.ldif`
```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to * by group/groupOfNames/member="cn=ldapadmin,ou=groups,dc=example,dc=com" manage by * read
```

#### 執行命令
```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f set_acl.ldif
```

**說明**：
- `manage` 權限賦予完全控制。
- `{0}` 表示最高優先級。

---

### 步驟 5（補遺）：設定 `sudoRole` 結構

為了支援 `sudo-ldap`，需引入 `sudo.schema` 並建立 `sudoRole` 條目，與 `cn=ldapadmin,ou=groups` 群組關聯。

#### 子步驟 5.1：引入 `sudo.schema`
若尚未導入，需加入 schema：

##### LDIF 文件：`sudo_schema.ldif`
```ldif
dn: cn=sudo,cn=schema,cn=config
changetype: add
objectClass: olcSchemaConfig
cn: sudo
olcSchemaFile: /etc/openldap/schema/sudo.schema
```

##### 執行命令
```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f sudo_schema.ldif
```

**注意**：
- 需先將 `sudo.schema` 放置在容器內 `/etc/openldap/schema/` 路徑。
- 可用 `wget https://raw.githubusercontent.com/openldap/openldap/master/contrib/slapd-modules/sudo/schema/sudo.schema` 下載。

#### 子步驟 5.2：建立 `sudoRole` 條目

##### LDIF 文件：`sudo_role.ldif`
```ldif
# 建立 ou=sudoers
dn: ou=sudoers,dc=example,dc=com
objectClass: organizationalUnit
ou: sudoers

# 定義 sudoRole 給 ldapadmin 群組
dn: cn=admins,ou=sudoers,dc=example,dc=com
objectClass: sudoRole
cn: admins
sudoUser: %ldapadmin  # 對應 cn=ldapadmin,ou=groups
sudoHost: ALL
sudoCommand: ALL
```

##### 執行命令
```bash
ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f sudo_role.ldif
```

**說明**：
- `sudoUser: %ldapadmin` 表示 `cn=ldapadmin,ou=groups` 的成員擁有 `sudo` 權限。
- `sudoHost: ALL` 和 `sudoCommand: ALL` 賦予完全權限，可根據需求調整。

---

### 步驟 6：驗證設定

1. **檢查結構**：
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "dc=example,dc=com" "(objectClass=*)"
```

2. **檢查 `memberOf`**：
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "dc=example,dc=com" "(member=cn=admin,dc=example,dc=com)" memberOf
```

3. **檢查 `sudoRole`**：
```bash
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "ou=sudoers,dc=example,dc=com" "(objectClass=sudoRole)"
```

---

### 注意事項
- **容器啟動**（以 `osixia/openldap` 為例）：
```bash
docker run -d --name openldap -p 389:389 -p 636:636 \
  -e LDAP_ORGANISATION="Example" \
  -e LDAP_DOMAIN="example.com" \
  -e LDAP_ADMIN_PASSWORD="admin" \
  osixia/openldap:latest
```
進入容器：
```bash
docker exec -it openldap bash
```
- **密碼安全性**：建議使用 `slappasswd` 生成加密密碼。
- **DN 調整**：替換 `dc=example,dc=com` 為你的實際 DN。
- **`sudoRole` 客戶端配置**：需搭配 RHEL 的 `sudo-ldap.conf` 和 NSS 配置使用。

---

### 總結
這些 LDIF 文件和命令完成了：
1. `memberOf` 啟用。
2. `ou=people` 和 `ou=groups` 建立。
3. `cn=ldapadmin,ou=groups` 建立並透過 ACL 賦予管理權限。
4. `sudoRole` 結構建立，與 `cn=ldapadmin,ou=groups` 整合，提供 `sudo` 權限。
```

---

### 保存與使用
你可以將上述內容複製到文字編輯器，保存為 `openldap_setup.md`，然後上傳到你的 GitHub 儲存庫，覆蓋或更新原有文件。如果需要進一步調整（例如添加更多範例用戶或群組），請告訴我！
