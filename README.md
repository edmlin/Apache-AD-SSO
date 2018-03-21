# Apache-AD-SSO
## Minimum setup for Apache+AD SSO  
参照：  

http://www.grolmsnet.de/kerbtut/

https://docs.typo3.org/typo3cms/extensions/ig_ldap_sso_auth/2.1.1/AdministratorManual/ConfigureApacheKerberos.html

#### 1. 安装软件包
```
yum install httpd  
yum install php  
yum install krb5-devel krb5-libs krb5-workstation mod_auth_kerb  
```
#### 2. 生成keytab文件

On DC:  
    ktpass /out webserver.keytab /princ HTTP/web.smallbusiness1.local@SMALLBUSINESS1.LOCAL /mapuser smallbusiness1\webauth /pass Test1234 /ptype KRB5_NT_PRINCIPAL /crypto rc4-hmac-nt

#### 3. 把keytab文件copy到/etc/httpd/

#### 4. DNS建立A记录和PTR记录指向web server

#### 5. In /etc/krb5.conf
```
    [libdefaults]  
    default_keytab_name = /etc/httpd/webserver.keytab  
    default_tkt_enctypes = rc4-hmac  
    default_tgs_enctypes = rc4-hmac  
    default_realm = SMALLBUSINESS1.LOCAL  

    [realms]  
    SMALLBUSINESS1.LOCAL = {  
    kdc = dc.smallbusiness1.local  
    admin_server = dc.smallbusiness1.local  
    }  

    [domain_realm]  
    .smallbusiness1.local = SMALLBUSINESS1.LOCAL  
    smallbusiness1.local = SMALLBUSINESS1.LOCAL  
```
 （注意：SMALLBUSINESS1.LOCAL需要大写）

#### 6. 测试（注意：SMALLBUSINESS1.LOCAL需要大写）：

    kinit user@SMALLBUSINESS1.LOCAL

    klist

#### 7. Apache配置

In /etc/httpd/conf/httpd.conf:

    ServerName web.smallbusiness1.local:80  
    UseCanonicalName On  

In /etc/httpd/conf.modules.d/10-auth_kerb.conf:

    LoadModule auth_kerb_module modules/mod_auth_kerb.so  
    <Location />  
    AuthType Kerberos  
    AuthName "Kerberos Login"  
    KrbMethodNegotiate On  
    KrbMethodK5Passwd On  
    KrbAuthRealms SMALLBUSINESS1.LOCAL  
    Krb5KeyTab /etc/httpd/webserver.keytab  
    KrbSaveCredentials On  
    require valid-user  
    </Location>  

#### 8. IE设置

Internet Option->Security->Local intranet->Sites->Add web.smallbusiness1.local  

Internet Option->Security->Local intranet->Custom Level->User Authentication->Logon->Automatic logon only in Intranet zone

#### 9. 打开IE，打开web.smallbusiness1.local/phpinfo.php.（注意，不能用IP地址）

#### Note:

1. Web server的时间要和DC的时间一致。

2. 如果error_log中看到gss_acquire_cred() ... (, Permission denied)，表示apache不能读取keytab文件，检查keytab文件权限，关闭selinux或者restorecon -rv /etc/httpd （keytab所在目录）

 

### Update: 支持多个domain

#### 1. 在每个domain的DC分别生成webserver1.keytab和webserver2.keytab，注意两个命令中HTTP/web.smallbusiness1.local是一样地，对应httpd.conf中的ServerName:

在smallbusiness1.local的DC:  
ktpass /out webserver1.keytab /princ HTTP/web.smallbusiness1.local@SMALLBUSINESS1.LOCAL /mapuser smallbusiness1\webauth /pass Test1234 /ptype KRB5_NT_PRINCIPAL /crypto rc4-hmac-nt

在smallbusiness2.local的DC:  
ktpass /out webserver2.keytab /princ HTTP/web.smallbusiness1.local@SMALLBUSINESS2.LOCAL /mapuser smallbusiness2\webauth /pass Test1234 /ptype KRB5_NT_PRINCIPAL /crypto rc4-hmac-nt

 

#### 2. 用ktutil合并keytab文件：
```
    ktutil  
    rkt webserver1.keytab  
    rkt webserver2.keytab  
    wkt webserver.keytab  
    q
```
用ktlist -k webserver.keytab 验证webserver.keytab中包含了多个key。

#### 3. 修改krb5.conf中的[realms]和[domain_realm]：
```
    [realms]  
    SMALLBUSINESS1.LOCAL = {  
    kdc = dc.smallbusiness1.local  
    admin_server = dc.smallbusiness1.local  
    }

    SMALLBUSINESS2.LOCAL = {  
    kdc = dc.smallbusiness2.local  
    admin_server = dc.smallbusiness2.local  
    }

    [domain_realm]  
    .smallbusiness1.local = SMALLBUSINESS1.LOCAL  
    smallbusiness1.local = SMALLBUSINESS1.LOCAL  

    .smallbusiness2.local = SMALLBUSINESS2.LOCAL  
    smallbusiness2.local = SMALLBUSINESS2.LOCAL  
```
#### 4. In /etc/httpd/conf.modules.d/10-auth_kerb.conf:
```
    LoadModule auth_kerb_module modules/mod_auth_kerb.so  
    <Location />  
    AuthType Kerberos  
    AuthName "Kerberos Login"  
    KrbMethodNegotiate On  
    KrbMethodK5Passwd On  
    KrbAuthRealms SMALLBUSINESS1.LOCAL SMALLBUSINESS2.LOCAL  
    Krb5KeyTab /etc/httpd/webserver.keytab  
    KrbSaveCredentials On  
    require valid-user  
    </Location>
```
