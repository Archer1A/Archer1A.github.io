# CentOS keystone 搭建
## 安装前准备
keystone 的版本号首字母为A-Z的单词，首字母越大版本越新

### 启用OpenStack repository
---

+ 在CentOS中 extras repository 提供了启用 OpenStack repository的RPM。 CentOS 默认包含了 extras repository,我们可以简单的安装就可以启用 OpenStack repository.
**安装Rocky版本**

```bash
    $ yum install centos-release-openstack-rocky
```

**安装Queens版本**

```bash
    $ yum install centos-release-openstack-queens
```

**安装pike版本**

```bash
    $ yum install centos-release-openstack-pike
```

### 安装完成

1. 更新包

```bash
    $  yum upgrade
```

2. 安装OpenStack client


```bash
    $  yum install python-openstackclient
```

3. CentOS 默认启动SELinux，安装**openstack-selinux**来自动管理OpenStack 服务的安全策略

```bash
    $ yum install openstack-selinux
```

### 创建数据库

安装部署前必须创建数据库
1. 使用root用户登录数据库

```bash
    $ mysql -uroot -p
```
2. 创建名字为**keystone**的数据库
```SQL
    MariaDB [(none)]> CREATE DATABASE keystone;
```
3. 创建登录用户

```bash
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%'  IDENTIFIED BY 'KEYSTONE_DBPASS';
```
替换**KEYSTONE_DBPASS** 为合适的密码

## 安装部署keystone
1. 运行以下命令安装keystone包:

```bash
    $  yum install openstack-keystone httpd mod_wsgi
```

2. 修改配置文件 `/etc/keystone/keystone.conf`

    + 在 **[database]** 项添加数据连接url：
    ```properties
     [database]
     #....
     connection = mysql://keystone:KEYSTONE_DBPASS@8.16.0.24/keystone
    ```
    **KEYSTONE_DBPASS**替换为上面配置的密码
    
    + 在 **[token]** 项配置Fernet token
    
    ```properties
    [token]
    #...
    provider = fernet
    ```
3. 配置keystone数据库：

```bash
    $  su -s /bin/sh -c "keystone-manage db_sync" keystone
```
4. 初始化Fernet key仓库

```bash
    $ keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    $ keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
5. 启动keystone ：

```bash
    $ keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
        --bootstrap-admin-url http://keystone:5000/v3/ \
        --bootstrap-internal-url http://keystone:5000/v3/ \
        --bootstrap-public-url http://keystone:5000/v3/ \
        --bootstrap-region-id RegionOne
```
替换 **ADMIN_PASS**为管理用户的密码

## 配置Apache HTTP server

1. 修改配置文件`/etc/httpd/conf/httpd.conf`中 **ServerName**字段的值

```properties
    ServerName keystone
```

2. 给`/usr/share/keystone/wsgi-keystone.conf`创建软链接
```bash
    $ ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

3. 启动apache  http服务并设置开机自启
```bash
    $ systemctl enable httpd.service
    $ systemctl start httpd.service
```

4. 给管理账号配置环境变量
```bash
    $ export OS_USERNAME=admin
    $ export OS_PASSWORD=ADMIN_PASS
    $ export OS_PROJECT_NAME=admin
    $ export OS_USER_DOMAIN_NAME=Default
    $ export OS_PROJECT_DOMAIN_NAME=Default
    $ export OS_AUTH_URL=http://keystone:5000/v3
    $ export OS_IDENTITY_API_VERSION=3
```
