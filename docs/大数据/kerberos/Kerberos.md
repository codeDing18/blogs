# Kerberos简介

## 环境信息

使用CentOS7操作系统。Ubuntu下的kerberos操作命令可能不同，以下均以CentOS7环境的为准。

## Kerberos的几个概念

### Realm

类似于namespace的概念，一个realm包含多个principal。一个principal属于一个特定的realm。

### Principal

认证的主体，可以认为等效于用户名。

Principal的名称格式为

```bash
name/role@realm
```

### Keytab

二进制文件。包含了principal和加密了的principal密钥信息，可以用来认证principal。

### Kadmin

Kadmin即Kerberos administration server，运行在主kerberos节点。负责存储KDC数据库，管理principal信息。

# Kerberos安装和配置

## 安装kerberos

Kerberos主节点（Kadmin，KDC）执行如下命令：

```shell
yum install -y krb5-server krb5-libs krb5-workstation
```

Kerberos从节点（只使用Kerberos认证）执行如下命令：

```shell
yum install -y krb5-devel krb5-workstation
```

## 配置krb5.conf

`krb5.conf`位于`/etc/krb5.conf`。

```csharp
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
# default_realm = EXAMPLE.COM
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
# EXAMPLE.COM = {
#  kdc = kerberos.example.com
#  admin_server = kerberos.example.com
# }

[domain_realm]
# .example.com = EXAMPLE.COM
# example.com = EXAMPLE.COM
```

logging模块：

配置默认，KDC和kadmin服务的log文件路径。

libdefaults模块：

- dns_lookup_realm：使用主机域名到kerberos domain的映射定位KDC。
- ticket_lifetime：ticket过期时间，超过这个时间ticket需要重新申请或renew。
- renew_lifetime：ticket可进行renew的时间限制。
- forwardable：如果配置为true，在KDC允许的情况下，初始ticket可以被转发。
- rdns：是否可使用逆向DNS。
- pkinit_anchors：签署KDC证书的根证书。
- default_realm：默认的realm。
- default_ccache_name：默认凭据缓存的命名规则。

realms模块：

使用如下的模版配置：

```undefined
EXAMPLE.COM = {
 kdc = kerberos.example.com
 admin_server = kerberos.example.com
}
```

- admin_server：kadmin服务（即Kerberos administration server）所在节点。
- kdc：KDC服务所在节点。

domain_realm模块：

此模块配置了domain name或者hostname同kerberos realm之间的映射关系。

官网配置项详细解释参见：[http://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html](https://links.jianshu.com/go?to=http%3A%2F%2Fweb.mit.edu%2Fkerberos%2Fkrb5-latest%2Fdoc%2Fadmin%2Fconf_files%2Fkrb5_conf.html)

## 配置kdc.conf

`kdc.conf`位于`/var/kerberos/krb5kdc/kdc.conf`。默认`kdc.conf`文件如下所示：

```ruby
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 EXAMPLE.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

- kdc_ports：KDC服务监听的端口。
- acl_file：ACL文件的路径。Kerberos使用这个ACL文件来确定哪些principal具有哪些权限。
- dict_file：存放一个由多行字符串构成的文本文件，该文件中的字符串禁止作为密码使用。
- admin_keytab：KDC 进行校验的 keytab。
- supported_enctypes：支持的加密算法类型。
- default_principal_flags：默认的principal标识，即创建principal时候无需特殊指定默认自带的标识。

官网配置项详细解释参见：[http://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/kdc_conf.html](https://links.jianshu.com/go?to=http%3A%2F%2Fweb.mit.edu%2Fkerberos%2Fkrb5-latest%2Fdoc%2Fadmin%2Fconf_files%2Fkdc_conf.html)

## 配置kadm5.acl

ACL文件用于控制kadmin数据库的访问权限，以及哪些principal可以操作其他的principal。位于`/var/kerberos/krb5kdc/kadm5.acl`。配置文件格式为：

```css
principal  permissions  [target_principal  [restrictions] ]
```

permissions官网有详细的列表，平时最为常用的是”*“，表示允许所有权限，并将该权限赋予管理员类型的principal。

例如我们配置：

```dart
*/admin@PAUL.COM    *
```

表示所有后缀为`/admin@PAUL.COM`的principal具有所有权限，充当管理员角色。

官网配置项详细解释参见：[http://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/kadm5_acl.html](https://links.jianshu.com/go?to=http%3A%2F%2Fweb.mit.edu%2Fkerberos%2Fkrb5-latest%2Fdoc%2Fadmin%2Fconf_files%2Fkadm5_acl.html)

## 初始化Kadmin数据库

命令格式为：

```shell
kdb5_util create -s -r [realm]
```

例如我们使用的realm为`PAUL.COM`，初始化数据库的命令为：

```shell
kdb5_util create -s -r PAUL.COM
```

根据提示输入database密码：

```kotlin
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'PAUL.COM',
master key name 'K/M@PAUL.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key:
Re-enter KDC database master key to verify:
```

## 启动Kerberos服务

```shell
systemctl start kadmin krb5kdc
```

# Kerberos操作

## Kadmin数据库操作

在运行kadmin的节点上执行如下命令，进入kadmin操作模式：

```shell
kadmin.local
```

> 如果有访问 KDC 服务器的 root 权限，但没有 kerberos admin 账户，使用 `kadmin.local`。
>
> 如果没有访问 KDC服务器的 root 权限，但用 kerberos admin 账户，使用 `kadmin`。
>
> 还可以使用`kadmin.local -q "命令"`的方式直接从shell操作kadmin数据库。

输入"?"可以获取到所有命令和解释：

```shell
kadmin.local:  ?
Available kadmin.local requests:

add_principal, addprinc, ank
                         Add principal
delete_principal, delprinc
                         Delete principal
modify_principal, modprinc
                         Modify principal
rename_principal, renprinc
                         Rename principal
change_password, cpw     Change password
get_principal, getprinc  Get principal
list_principals, listprincs, get_principals, getprincs
                         List principals
add_policy, addpol       Add policy
modify_policy, modpol    Modify policy
delete_policy, delpol    Delete policy
get_policy, getpol       Get policy
list_policies, listpols, get_policies, getpols
                         List policies
get_privs, getprivs      Get privileges
ktadd, xst               Add entry(s) to a keytab
ktremove, ktrem          Remove entry(s) from a keytab
lock                     Lock database exclusively (use with extreme caution!)
unlock                   Release exclusive database lock
purgekeys                Purge previously retained old keys from a principal
get_strings, getstrs     Show string attributes on a principal
set_string, setstr       Set a string attribute on a principal
del_string, delstr       Delete a string attribute on a principal
list_requests, lr, ?     List available requests.
quit, exit, q            Exit program.
```

### listprincs

列出所有的principal。

```shell
kadmin.local:  listprincs
K/M@PAUL.COM
kadmin/admin@PAUL.COM
kadmin/changepw@PAUL.COM
kadmin/d7b07e9f1287@PAUL.COM
kiprop/d7b07e9f1287@PAUL.COM
krbtgt/PAUL.COM@PAUL.COM
```

### addprinc

添加一个principal。如果没有指定`-randkey`或`-nokey`参数，需要指定一个密码。

```shell
kadmin.local:  addprinc demo/localhost
WARNING: no policy specified for demo/localhost@PAUL.COM; defaulting to no policy
Enter password for principal "demo/localhost@PAUL.COM":
Re-enter password for principal "demo/localhost@PAUL.COM":
Principal "demo/localhost@PAUL.COM" created.
```

此时可以使用kinit命令，登陆这个principal。

```shell
sh-4.2# kinit demo/localhost@PAUL.COM
Password for demo/localhost@PAUL.COM:
sh-4.2# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: demo/localhost@PAUL.COM

Valid starting     Expires            Service principal
03/23/21 01:26:55  03/24/21 01:26:55  krbtgt/PAUL.COM@PAUL.COM
```

### modprinc

修改principal。为principal增加或去掉部分属性。包含的属性和参数参见命令帮助。

```shell
kadmin.local:  modprinc
usage: modify_principal [options] principal
        options are:
                [-x db_princ_args]* [-expire expdate] [-pwexpire pwexpdate] [-maxlife maxtixlife]
                [-kvno kvno] [-policy policy] [-clearpolicy]
                [-maxrenewlife maxrenewlife] [-unlock] [{+|-}attribute]
        attributes are:
                allow_postdated allow_forwardable allow_tgs_req allow_renewable
                allow_proxiable allow_dup_skey allow_tix requires_preauth
                requires_hwauth needchange allow_svr password_changing_service
                ok_as_delegate ok_to_auth_as_delegate no_auth_data_required
                lockdown_keys

where,
        [-x db_princ_args]* - any number of database specific arguments.
                        Look at each database documentation for supported arguments
```

### delprinc

删除principal。

```shell
kadmin.local:  delprinc test/localhost
Are you sure you want to delete the principal "test/localhost@PAUL.COM"? (yes/no): yes
Principal "test/localhost@PAUL.COM" deleted.
Make sure that you have removed this principal from all ACLs before reusing.
```

### change_password

修改principal的密码。之后使用`kinit`命令认证，需要使用新的密码。

```shell
kadmin.local:  change_password demo/localhost@PAUL.COM
Enter password for principal "demo/localhost@PAUL.COM":
Re-enter password for principal "demo/localhost@PAUL.COM":
Password for "demo/localhost@PAUL.COM" changed.
```

### ktadd

生成一个keytab，或者是将一个principal加入到keytab。

```shell
kadmin.local:  ktadd -norandkey -k /root/demo.keytab demo/localhost@PAUL.COM
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type aes256-cts-hmac-sha1-96 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type aes128-cts-hmac-sha1-96 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type des3-cbc-sha1 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type arcfour-hmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type camellia256-cts-cmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type camellia128-cts-cmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type des-hmac-sha1 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo/localhost@PAUL.COM with kvno 4, encryption type des-cbc-md5 added to keytab WRFILE:/root/demo.keytab.
```

对于上面这条命令，如果执行的时候`/root/demo.keytab`不存在，会生成一个新的keytab文件。然后将`demo/localhost@PAUL.COM`这个principal添加到该keytab。`-norandkey`参数的含义是不更改密码。也就是说现在这个principal既可以使用原来的密码认证，也可以使用新生成的keytab认证。

> 我们在`kdc.conf`的`supported_enctypes`配置项指定了8种加密算法，因此这里会打印出8个entry。

使用keytab方式认证的命令如下：

```shell
kinit -kt demo.keytab demo/localhost@PAUL.COM
```

我们可以使用`ktadd`命令，将多个principal加入同一个keytab文件，这样该keytab文件可用于认证多个用户。例如：

```shell
kadmin.local:  addprinc test/localhost@PAUL.COM
WARNING: no policy specified for test/localhost@PAUL.COM; defaulting to no policy
Enter password for principal "test/localhost@PAUL.COM":
Re-enter password for principal "test/localhost@PAUL.COM":
Principal "test/localhost@PAUL.COM" created.
kadmin.local:  ktadd -kt /root/demo.keytab test/localhost@PAUL.COM
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type aes256-cts-hmac-sha1-96 added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type aes128-cts-hmac-sha1-96 added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type des3-cbc-sha1 added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type arcfour-hmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type camellia256-cts-cmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type camellia128-cts-cmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type des-hmac-sha1 added to keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4, encryption type des-cbc-md5 added to keytab WRFILE:/root/demo.keytab.
```

此时我们使用`klist`命令查看下关联了`/root/demo.keytab`文件的principal：

```shell
sh-4.2# klist -kt demo.keytab
Keytab name: FILE:demo.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
   4 03/23/21 05:57:39 test/localhost@PAUL.COM
```

看到这个输出说明`demo.keytab`已经关联这两个principal。

### ktremove

从keytab中删除关联的principal。

接着上面的例子，如果需要删除`test/localhost@PAUL.COM`和`/root/demo.keytab`的关联，执行如下命令：

```shell
kadmin.local:  ktremove -k /root/demo.keytab test/localhost@PAUL.COM
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
Entry for principal test/localhost@PAUL.COM with kvno 4 removed from keytab WRFILE:/root/demo.keytab.
```

然后我们使用`klist`查看`/root/demo.keytab`关联的principal：

```shell
sh-4.2# klist -kt demo.keytab
Keytab name: FILE:demo.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
```

发现`test/localhost@PAUL.COM`的8个entry已经被移除。我们无法再使用`/root/demo.keytab`认证`test/localhost@PAUL.COM`。

## Kerberos命令

### kinit

获取principal授予的票据，并缓存（认证principal）。

可以使用`-h`参数获取该命令的帮助信息：

```shell
kinit -h
```

#### 使用password进行认证

直接输入`kinit principal`，然后命令行会提示输入密码。

```shell
sh-4.2# kinit demo/localhost@PAUL.COM
Password for demo/localhost@PAUL.COM:
sh-4.2# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: demo/localhost@PAUL.COM

Valid starting     Expires            Service principal
03/23/21 01:26:55  03/24/21 01:26:55  krbtgt/PAUL.COM@PAUL.COM
```

#### 使用keytab进行认证

和password不同的是，我们使用`-kt`参数指定keytab文件的路径，例如：

```shell
kinit demo/localhost@PAUL.COM -kt /root/demo.keytab
```

#### Ticket续约

如果没有配置KDC允许续约，会出现类似如下问题：

执行`klist`，没有续约提示。

```shell
sh-4.2# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: demo/localhost@PAUL.COM

Valid starting     Expires            Service principal
03/23/21 02:46:06  03/24/21 02:46:06  krbtgt/PAUL.COM@PAUL.COM
```

执行`kinit -R`，报如下错误：

```shell
sh-4.2# kinit -R
kinit: KDC can't fulfill requested option while renewing credentials
```

解决方法：

编辑`/var/kerberos/krb5kdc/kdc.conf`文件，按照如下注释修改配置：

```ruby
[realms]
 PAUL.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  # 增加最大允许续约时间
  max_renewable_life = 7d 0h 0m 0s
  # 增加principal默认的flag：允许续约
  default_principal_flags = +renewable
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

然后重启Kerberos相关服务，重新创建`kadmin`数据库：

```shell
systemctl restart kadmin krb5kdc
```

如果`kadmin`数据库已存在，使用下方命令删除：

```shell
sh-4.2# kdb5_util destroy -r PAUL.COM
Deleting KDC database stored in '/var/kerberos/krb5kdc/principal', are you sure?
(type 'yes' to confirm)? yes
OK, deleting database '/var/kerberos/krb5kdc/principal'...
** Database '/var/kerberos/krb5kdc/principal' destroyed.
```

再创建数据库：

```shell
kdb5_util create -s -r PAUL.COM
```

然后使用`addprinc`等命令创建principal和keytab。

```shell
kadmin.local:  addprinc demo
WARNING: no policy specified for demo@PAUL.COM; defaulting to no policy
Enter password for principal "demo@PAUL.COM":
Re-enter password for principal "demo@PAUL.COM":
Principal "demo@PAUL.COM" created.
kadmin.local:  ktadd -kt /root/demo.keytab demo
Entry for principal demo with kvno 2, encryption type aes256-cts-hmac-sha1-96 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type aes128-cts-hmac-sha1-96 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type des3-cbc-sha1 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type arcfour-hmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type camellia256-cts-cmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type camellia128-cts-cmac added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type des-hmac-sha1 added to keytab WRFILE:/root/demo.keytab.
Entry for principal demo with kvno 2, encryption type des-cbc-md5 added to keytab WRFILE:/root/demo.keytab.
```

然后我们使用`kinit`命令认证，并使用`klist`命令查看：

```shell
sh-4.2# kinit -kt demo.keytab demo
sh-4.2# klist
Ticket cache: KEYRING:persistent:0:krb_ccache_vhzDpIA
Default principal: demo@PAUL.COM

Valid starting     Expires            Service principal
03/23/21 08:31:43  03/24/21 08:31:43  krbtgt/PAUL.COM@PAUL.COM
        renew until 03/30/21 08:31:43
```

我们发现`klist`输出多了`rennew until`字样，表示在这个日期前可以续约。执行`kinit -R`命令续约：

```shell
sh-4.2# kinit -R
sh-4.2# klist
Ticket cache: KEYRING:persistent:0:krb_ccache_vhzDpIA
Default principal: demo@PAUL.COM

Valid starting     Expires            Service principal
03/23/21 08:33:56  03/24/21 08:33:56  krbtgt/PAUL.COM@PAUL.COM
        renew until 03/30/21 08:31:43
```

此时`kinit -R`命令不再报错，且`Valid starting`和`Expires`时间已经更新。

注意：

如果我们已经创建出的principal不允许续约或者是更改最大允许续约时间，可执行如下命令：

```shell
modprinc -maxrenewlife 1week +allow_renewable demo/localhost@PAUL.COM
```

### kdestroy

销毁当前认证票据，删除凭据缓存。该命令不需要任何参数。可使用`kdestroy -A`清除所有凭据缓存。

### klist

查看当前凭据缓存内的票据。

```shell
sh-4.2# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: demo/localhost@PAUL.COM

Valid starting     Expires            Service principal
03/23/21 01:26:55  03/24/21 01:26:55  krbtgt/PAUL.COM@PAUL.COM
```

如果处于未认证状态，返回的结果如下所示：

```shell
sh-4.2# klist
klist: Credentials cache keyring 'persistent:0:0' not found
```

除此之外klist命令还可以列出某个keytab文件关联的principal。

```shell
sh-4.2# klist -kt demo.keytab
Keytab name: FILE:demo.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
   4 03/23/21 05:38:02 demo/localhost@PAUL.COM
```

## ktutil命令

`ktutil`是方便用户操作keytab文件的工具集。我们可以使用`ktutil`命令进入`ktutil`模式。

键入"?"获取`ktutil`的帮助信息。

```shell
ktutil:  ?
Available ktutil requests:

clear_list, clear        Clear the current keylist.
read_kt, rkt             Read a krb5 keytab into the current keylist.
read_st, rst             Read a krb4 srvtab into the current keylist.
write_kt, wkt            Write the current keylist to a krb5 keytab.
write_st, wst            Write the current keylist to a krb4 srvtab.
add_entry, addent        Add an entry to the current keylist.
delete_entry, delent     Delete an entry from the current keylist.
list, l                  List the current keylist.
list_requests, lr, ?     List available requests.
quit, exit, q            Exit program.
```

`ktutil`命令常用于合并keytab文件，比如我们有：

- `/root/demo.keytab` 对应 `demo/localhost@PAUL.COM`
- `/root/test.keytab` 对应 `test/localhost@PAUL.COM`

我们可以用如下命令将这两个keytab合并为`/root/merged.keytab`：

```shell
ktutil:  rkt demo.keytab
ktutil:  rkt test.keytab
ktutil:  l
slot KVNO Principal
---- ---- ---------------------------------------------------------------------
   1    4                  demo/localhost@PAUL.COM
   2    4                  demo/localhost@PAUL.COM
   3    4                  demo/localhost@PAUL.COM
   4    4                  demo/localhost@PAUL.COM
   5    4                  demo/localhost@PAUL.COM
   6    4                  demo/localhost@PAUL.COM
   7    4                  demo/localhost@PAUL.COM
   8    4                  demo/localhost@PAUL.COM
   9    5                  test/localhost@PAUL.COM
  10    5                  test/localhost@PAUL.COM
  11    5                  test/localhost@PAUL.COM
  12    5                  test/localhost@PAUL.COM
  13    5                  test/localhost@PAUL.COM
  14    5                  test/localhost@PAUL.COM
  15    5                  test/localhost@PAUL.COM
  16    5                  test/localhost@PAUL.COM
ktutil: wkt /root/merged.keytab
```

到此为止`/root/merged.keytab`文件可用于认证这两个principal。我们可以用`klist`命令查看下：

```shell
sh-4.2# klist -kt merged.keytab
Keytab name: FILE:merged.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   4 03/23/21 06:55:13 demo/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
   5 03/23/21 06:55:13 test/localhost@PAUL.COM
```

## Docker搭建Kerberos开发环境

我们基于CentOS7，创建Kerberos镜像。编写`Dockerfile`如下：

```dockerfile
FROM centos:centos7
RUN yum install -y krb5-server krb5-libs krb5-workstation
CMD ["/usr/sbin/init"]
```

然后执行如下命令，构建镜像：

```shell
docker build -t kerberos:0.1 .
```

启动该Kerberos容器的命令：

```shell
docker run --privileged -p 88:88 -p 749:749 -p 750:750 -d --name=kerberos kerberos:0.1
```

> 注意，必须添加`--privileged`参数，且程序入口为`/usr/sbin/init`。只有这样才能够在容器内运行`systemctl`命令，否则会出错。

进入容器的方法：

```shell
docker exec -it kerberos sh
```

然后我们可以像真机环境一样操作Kerberos了。



# 参考

[Kerberos 安装和使用](https://www.jianshu.com/p/032cc462bbca)

[原理](https://zhuanlan.zhihu.com/p/266491528)

[[Kerberos基本原理、安装部署及用法](https://www.cnblogs.com/swordfall/p/12009716.html)](https://www.cnblogs.com/swordfall/p/12009716.html)

