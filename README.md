# ldap-util
部署ldap来维护linux账户的小工具，并且使用google authenticator做二步验证。

全文是基于example.com来举例说明，脚本出于偷懒，冗余了example，请sed来替换下。

## Requirements

- Erlang 19.3
- Ubuntu 14.04

## server

### 安装

```shell
shell/> ./install ## 从源码安装ldap
```

修改`/opt/ldap/etc/openldap/slapd.conf`，将suffix和rootdn的my-domain换成自己的域名，比如example

include所有schema 

```shell
shell/> for f in `ls /opt/ldap/etc/openldap/schema/*.schema`; do echo include $f; done # 打印
# 复制出来结果到/opt/ldap/etc/openldap/slapd.conf，注意：core要放第一行
```

```shell
shell/> ./start ## 启动
shell/> ldapsearch -x -b 'cn=Manager,dc=example,dc=com' 'objectclass=*' # 正常响应则启动成功
```

### 初始化数据

```shell
ldap-util-root/server> ./init-data # 根目录，账户和组的目录初始化
## 请配置ldap-util.conf再执行下面的
## 用户
ldap-util-root/server> ./ldap-util user_add roowe 66660001 66660 "ssh pub key" ## 新增用户
ldap-util-root/server> ./ldap-util user_del roowe ## 删除用户
ldap-util-root/server> ./ldap-util user_change_ga_secret roowe ## 更换狗牌
ldap-util-root/server> ./ldap-util user_add_pubkey roowe "ssh pub key" ## 新增ssh pub key
ldap-util-root/server> ./ldap-util user_del_pubkey roowe "ssh pub key" ## 传单个则删除指定的或者不传就清空

## 组
ldap-util-root/server> ./ldap-util group_add sa 66660 ## 新增组
ldap-util-root/server> ./ldap-util group_del sa ## 删除组
ldap-util-root/server> ./ldap-util group_add_member roowe sa saa ## 将用户roowe添加到组sa和saa
ldap-util-root/server> ./ldap-util group_del_member roowe sa ## 从组sa移除用户roowe

```

## agent

### 安装

```shell
shell/> ./ldap-client-install ## 执行前先打开修改自己的ldap server配置
```

```shell
ldap-util-root/agent> mkdir /sshd/ldap/
ldap-util-root/agent> cp getcn getpub-wapper /sshd/ldap/
```

### openssh配置修改

```
## /etc/ssh/sshd_config
AuthorizedKeysCommand /sshd/ldap/getpub-wrapper
AuthorizedKeysCommandUser nobody

## 以下是google authenticator的配置，不需要可以不添加
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive

## 登录组限制
AllowGroups sa
```

### google authenticator

```shell
shell/> sudo apt-get install libpam-google-authenticator ## 安装google-authenticator
shell/> mkdir /sshd/google_authenticator/
shell/> chown nobody:nogroup /sshd/google_authenticator/
```

```
## 文件/etc/pam.d/sshd配置
auth    required        pam_google_authenticator.so user=nobody secret=/sshd/google_authenticator/${USER}
#@include common-auth  #这一样必须屏蔽掉，不然老是会问你密码
```



## 最后

配置完成之后，agent会从server获取user和group信息，从而提供认证，这样一份配置，可以多机器使用。

```shell
ssh username@agent-ip
```



