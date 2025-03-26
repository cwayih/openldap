# OpenLDAP 設定問題與解答

## 提問
如果在空的 OpenLDAP server 容器要建立 `ou=people`、`ou=groups`，開啟 `memberof`，`cn=ldapadmin,ou=groups` 讓這個 `ldapadmin` 等效於 `cn=admin`，這樣的 LDIF 如何設定及執行？

## 回答

### 概述
以下說明如何在空的 OpenLDAP 伺服器容器中設定：
- 建立 `ou=people` 和 `ou=groups`。
- 啟用 `memberOf` 覆蓋模組。
- 建立 `cn=ldapadmin,ou=groups` 並賦予其與 `cn=admin` 等效的管理權限。

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
