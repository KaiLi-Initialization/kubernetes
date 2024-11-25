## zabbix安装

### 安装Mysql

#### 在RHEL/CentOS上

1. **添加MySQL仓库**:

   ```shell
   sudo rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
   ```

2. **安装MySQL服务器**:

   ```shell
   sudo yum install mysql-server
   ```

3. **启动MySQL服务**:

   ```shell
   sudo systemctl start mysqld
   ```

4. **找到临时root密码**:

   - 安装后MySQL会生成一个临时密码，可以在日志文件中找到。

   ```shell
   sudo grep 'temporary password' /var/log/mysqld.log
   ```

5. **配置MySQL**:

   - 使用临时密码登录MySQL并设置新密码。

   ```shell
   # 登录mysql
   mysql -u root -p
   # 设置用户和密码
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewStrongPassword';
   ```

6. **运行安全安装脚本**:

   ```shell
   sudo mysql_secure_installation
   ```

安装完成后，你可以使用MySQL命令行工具或图形化工具（如MySQL Workbench）来管理你的数据库。

### 安装zabbix

1. ##### 安装 Zabbix 存储库

   如果安装了 EPEL 提供的 Zabbix 包，请禁用它。编辑文件 /etc/yum.repos.d/epel.repo 并添加以下语句。

   ```shell
   [epel]...excludepkgs=zabbix*
   ```

   继续安装 zabbix 存储库。

   ```shell
   # rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-7.0-3.el9.noarch.rpm# dnf clean all
   ```

2. ##### 安装Zabbix服务器，前端，代理

   ```shell
   # dnf install zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent
   ```

   

3. ##### 创建初始数据库

   确保数据库服务器已启动并正在运行。

   在数据库主机上运行以下命令。

   ```shell
   # mysql -uroot -p
   password
   mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
   mysql> create user zabbix@localhost identified by 'password';
   mysql> grant all privileges on zabbix.* to zabbix@localhost;
   mysql> set global log_bin_trust_function_creators = 1;
   mysql> quit;
   ```

   在 Zabbix 服务器主机上导入初始模式和数据。系统将提示您输入新创建的密码。

   ```shell
   # zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
   ```

   导入数据库模式后禁用 log_bin_trust_function_creators 选项。

   ```shell
   # mysql -uroot -ppasswordmysql> set global log_bin_trust_function_creators = 0;mysql> quit;
   ```

4. ##### 为Zabbix服务器配置数据库

   编辑文件 /etc/zabbix/zabbix_server.conf

   ```shell
   DBPassword=password
   ```

5. ##### 为 Zabbix 前端配置 PHP

   编辑文件 /etc/nginx/conf.d/zabbix.conf 取消注释并设置“listen”和“server_name”指令。

   ```
   # listen 8080;# server_name example.com;
   ```

6. ##### 启动Zabbix服务器和代理进程

   启动 Zabbix 服务器和代理进程并使其在系统启动时启动。

   ```
   # systemctl restart zabbix-server zabbix-agent nginx php-fpm# systemctl enable zabbix-server zabbix-agent nginx php-fpm
   ```