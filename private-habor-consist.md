
# 一、安装docker以及准备工作
```
 此处可以参考我的k8s-kubeadm-install中docker安装的部分
```
## 1.1在hub中
```
# vim /etc/docker/daemon.json
  添加一行
  "insecure-registries": ["https://hub.jerry.com"]  ##后面这个域名，自己填写自己的就ok
```

# 二、安装Harbor

## 2.1 复制所需要的软件
```
将docker-compose 和harbor-offline-installer-v1.2.0.tgz复制到hub的/root/下
# mv docker-compose  /usr/local/bin/
# chmod a+x /usr/local/bin/docker-compose


# tar -zxvf harbor-offline-installer-v1.2.0.tgz
```

## 2.2 配置harbor.cfg

hostname：目标的主机名或者完全限定域名  
ui_url_protocol：http或https。默认为http  
db_password：用于db_auth的的MySQL数据库的根密码。更改此密码进行任何生产用途  
max_job_workers：(默认值为3)作业服务中的复制工作人员的最大数量，对于每个映像复制作业，工作人员将存储库的所有标签同步到远程目标。增加此数字允许系统中更多的并发复制作业。但是，由于每个工作人员都会消耗一定数量的网络/CPU/IO资源，请根据主机的硬件资源，仔细选择该属性的值  
customize_crt：（on或off。默认为on）当此属性打开时，prepare脚本将为注册表的令牌的生成/验证创验证创建私钥和根证书  
ssl_cert：：SSL证书的路径，，仅当协议设置为https时才应用  
ssl_cert_key：：SSL密钥的路径，仅当协议设置为https时才应用  
secretkey_path：用于在复制策略中加密或解密远程注册表的密码的密钥路径  
```
# mv harbor /usr/local/

# cd /usr/local/harbor/
# vim harbor.cfg
  ## Configuration file of Harbor

  #The IP address or hostname to access admin UI and registry service.
  #DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
  hostname = hub.jerry.com

  #The protocol for accessing the UI and token/notification service, by default it is http.
  #It can be set to https if ssl is enabled on nginx.
  ui_url_protocol = https

  #The password for the root user of mysql db, change this before any production use.
  db_password = root123

  #Maximum number of job workers in job service  
  max_job_workers = 3 

  #Determine whether or not to generate certificate for the registry's token.
  #If the value is on, the prepare script creates new root cert and private key 
  #for generating token to access the registry. If the value is off the default key/cert will be used.
  #This flag also controls the creation of the notary signer's cert.
  customize_crt = on

  #The path of cert and key files for nginx, they are applied only the protocol is set to https
  ssl_cert = /data/cert/server.crt
  ssl_cert_key = /data/cert/server.key

  #The path of secretkey storage
  secretkey_path = /data

  #Admiral's url, comment this attribute, or set its value to NA when Harbor is standalone
  admiral_url = NA

  #The password of the Clair's postgres database, only effective when Harbor is deployed with Clair.
  #Please update it before deployment, subsequent update will cause Clair's API server and Harbor unable to access Clair's database.
  clair_db_password = password

  #NOTES: The properties between BEGIN INITIAL PROPERTIES and END INITIAL PROPERTIES
  #only take effect in the first boot, the subsequent changes of these properties 
  #should be performed on web ui

  #************************BEGIN INITIAL PROPERTIES************************

  #Email account settings for sending out password resetting emails.

  #Email server uses the given username and password to authenticate on TLS connections to host and act as identity.
  #Identity left blank to act as username.
  email_identity = 

  email_server = smtp.mydomain.com
  email_server_port = 25
  email_username = sample_admin@mydomain.com
  email_password = abc
  email_from = admin <sample_admin@mydomain.com>
  email_ssl = false

  ##The initial password of Harbor admin, only works for the first time when Harbor starts. 
  #It has no effect after the first launch of Harbor.
  #Change the admin password from UI after launching Harbor.
  harbor_admin_password = Harbor12345

  ##By default the auth mode is db_auth, i.e. the credentials are stored in a local database.
  #Set it to ldap_auth if you want to verify a user's credentials against an LDAP server.
  auth_mode = db_auth

  #The url for an ldap endpoint.
  ldap_url = ldaps://ldap.mydomain.com

  #A user's DN who has the permission to search the LDAP/AD server. 
  #If your LDAP/AD server does not support anonymous search, you should configure this DN and ldap_search_pwd.
  #ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

  #the password of the ldap_searchdn
  #ldap_search_pwd = password

  #The base DN from which to look up a user in LDAP/AD
  ldap_basedn = ou=people,dc=mydomain,dc=com

  #Search filter for LDAP/AD, make sure the syntax of the filter is correct.
  #ldap_filter = (objectClass=person)

  # The attribute used in a search to match a user, it could be uid, cn, email, sAMAccountName or other attributes depending on your LDAP/AD  
  ldap_uid = uid 

  #the scope to search for users, 1-LDAP_SCOPE_BASE, 2-LDAP_SCOPE_ONELEVEL, 3-LDAP_SCOPE_SUBTREE
  ldap_scope = 3 

  #Timeout (in seconds)  when connecting to an LDAP Server. The default value (and most reasonable) is 5 seconds.
  ldap_timeout = 5

  #Turn on or off the self-registration feature
  self_registration = on

  #The expiration time (in minute) of token created by token service, default is 30 minutes
  token_expiration = 30

  #The flag to control what users have permission to create projects
  #The default value "everyone" allows everyone to creates a project. 
  #Set to "adminonly" so that only admin user can create project.
  project_creation_restriction = everyone

  #Determine whether the job service should verify the ssl cert when it connects to a remote registry.
  #Set this flag to off when the remote registry uses a self-signed or untrusted certificate.
  verify_remote_cert = on
##创建上面harbor.cfg中存放证书的文件夹
# mkdir -p /data/cert

# cd !$
# openssl genrsa -des3 -out server.key 2048
# openssl req -new -key server.key -out server.csr
# cp server.key server.key.org
# openssl rsa -in server.key.org -out server.key
# openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
# mkdir  /data/cert
# chmod -R 777 /data/cert

```

## 2.3 运行脚本安装harbor
```
# cd /usr/local/harbor
# ./install.sh 
```
## 2.4 访问测试
在windos的C:\Windows\System32\drivers\etc
```
打开hosts文件，添加如下内容 
192.168.133.65  hub.jerry.com

保存退出后，可以在浏览器输入hub.jerry.com进行访问
```
# 三、使用Harbor
## 3.1使用harbor节点配置 
```
# vim /etc/docker/daemon.json
  insecure-registries上面不要忘记加逗号，将再hub中新增的域名添加到daemon.json中即可
    {
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
  "max-size": "100m"
  }, 
  "insecure-registries": ["https://hub.jerry.com"]
  }
##重启docker
# systemctl restart docker
```
## 3.2 上传镜像
```
1、首先登陆Harbor
# docker login https://hub.jerry.com 

2、将本地的镜像打标签
#docker tag SOURCE_IMAGE[:TAG] hub.jerry.com/library/IMAGE[:TAG]

3、推送镜像到仓库
docker push hub.jerry.com/library/IMAGE[:TAG]

```
## 3.3 下载镜像

```
镜像地址为：hub.jerry.com/library/IMAGE[:TAG]
```
