1. 需要安装`Docker`，步骤如下：
  - ```sh
    curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
    sudo systemctl enable docker
    sudo systemctl start docker
    ```


2. 需要安装`docker-compose`，步骤如下：
  - ```sh
    sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    ```
    如果下载慢可使用加速地址

    ```sh
    sudo curl -L "https://download.fastgit.org/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    ```

  - ```sh
    sudo chmod +x /usr/local/bin/docker-compose
    ```

  - ```sh
    sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
    ```
  
  - ```sh
    docker-compose -v
    ```

3. 使用以下`shell`脚本完成数据库脚本的初始化（应和apollo的版本使用同一版本，目前都是`1.1.2`，针对不同版本的sql脚本需要微调，主要是`sed`操作的内容行数可能不同）

```sh
mkdir -p sql
curl -L "https://raw.fastgit.org/apolloconfig/apollo/v1.1.2/scripts/docker-quick-start/sql/apolloconfigdb.sql" -o ./sql/apolloconfigdb.sql
sed -i '373,402d' ./sql/apolloconfigdb.sql
cp ./sql/apolloconfigdb.sql ./sql/apolloconfigdb_dev.sql
sed -i 's/ApolloConfigDB/ApolloConfigDB_dev/g' ./sql/apolloconfigdb_dev.sql
cp ./sql/apolloconfigdb.sql ./sql/apolloconfigdb_fat.sql
sed -i 's/ApolloConfigDB/ApolloConfigDB_fat/g' ./sql/apolloconfigdb_fat.sql
sed -i 's/8080/8081/g' ./sql/apolloconfigdb_fat.sql
cp ./sql/apolloconfigdb.sql ./sql/apolloconfigdb_uat.sql
sed -i 's/ApolloConfigDB/ApolloConfigDB_uat/g' ./sql/apolloconfigdb_uat.sql
sed -i 's/8080/8082/g' ./sql/apolloconfigdb_uat.sql
cp ./sql/apolloconfigdb.sql ./sql/apolloconfigdb_pro.sql
sed -i 's/ApolloConfigDB/ApolloConfigDB_pro/g' ./sql/apolloconfigdb_pro.sql
sed -i 's/8080/8083/g' ./sql/apolloconfigdb_pro.sql
rm -f ./sql/apolloconfigdb.sql

curl -L "https://raw.fastgit.org/apolloconfig/apollo/v1.1.2/scripts/docker-quick-start/sql/apolloportaldb.sql" -o ./sql/apolloportaldb.sql
sed -i '323,359d' ./sql/apolloportaldb.sql
sed -i 's/dev/DEV,FAT,UAT,PRO/g' ./sql/apolloportaldb.sql
sed -i '311c \(\x27organizations\x27, \x27\[\{"orgId":"org1","orgName":"研发部"\}\]\x27, \x27部门列表\x27\),' ./sql/apolloportaldb.sql
sed -i 's/apollo@acme.com/yunwei@bakclass.com/g' ./sql/apolloportaldb.sql
```

4. 进入到`docker-compose.yaml`所在目录，根据工程中使用的`apollo`客户端版本修改镜像版本，修改各环境变量，可按需配置需要搭建的环境（注释不需要的环境的`DB`配置和`ports`配置）
  - 如果`network_mode = host`，会直接暴露容易使用的端口，`ports`会失效，需要使用环境变量修改暴露的端口
  - 默认情况下`network_mode = bridge`，可以使用`ports`配置对端口进行选择性配置
  - 根据`network_mode`的不同，数据库的`IP`地址会不一样

5. 执行`docker-compose up -d`

附件`docker-compose.yaml`：

```yaml
version: "2.2"
services:
  apollo:
    image: idoop/docker-apollo:1.1.2
    container_name: apollo
    # portal若出现504错误,则将网络模式改为host. host模式下如果想改端口,参考下方修改端口的环境变量
    # network_mode: host
    restart: always
    volumes:
      # 如果需要查看日志,挂载容器中的/opt路径出来即可.
      - ./logs:/opt
      # 如果portal需要开启ldap或ad域验证,须挂载此ldap配置文件
    #  - ./application-ldap.yml:/apollo-portal/config/application-ldap.yml:ro
    depends_on:
      - mysql
    links:
      - mysql
    environment:
      # 开启Portal实例,默认端口: 8070
      PORTAL_DB: jdbc:mysql://mysql:3306/ApolloPortalDB?characterEncoding=utf8
      PORTAL_DB_USER: root
      PORTAL_DB_PWD: 123456
      # PORTAL_PORT: 8070
      # 如果portal需要开启ldap或ad域验证,须设置该环境变量为TRUE
      #PORTAL_LDAP: "TRUE"

      # 开启dev环境实例, 默认端口: config 8080, admin 8090
      DEV_DB: jdbc:mysql://mysql:3306/ApolloConfigDB_dev?characterEncoding=utf8
      DEV_DB_USER: root
      DEV_DB_PWD: 123456
      # 若network_mode为host,则可修改端口,如下:
      # DEV_CONFIG_PORT: 8080
      # DEV_ADMIN_PORT: 8090

      # 开启fat环境实例, 默认端口: config 8081, admin 8091
      FAT_DB: jdbc:mysql://mysql:3306/ApolloConfigDB_fat?characterEncoding=utf8
      FAT_DB_USER: root
      FAT_DB_PWD: 123456
      # 若network_mode为host,则可修改端口,如下:
      # FAT_CONFIG_PORT: 8081
      # FAT_ADMIN_PORT: 8091

      # uat环境实例, 默认端口: config 8082, admin 8092
      UAT_DB: jdbc:mysql://mysql:3306/ApolloConfigDB_uat?characterEncoding=utf8
      UAT_DB_USER: root
      UAT_DB_PWD: 123456
      # 若network_mode为host,则可修改端口,如下:
      # UAT_CONFIG_PORT: 8082
      # UAT_ADMIN_PORT: 8092

      # 给Portal指定远程uat地址
      #UAT_URL: http://192.168.1.2:8082

      # pro环境实例, 默认端口: config 8083, admin 8093
      PRO_DB: jdbc:mysql://mysql:3306/ApolloConfigDB_pro?characterEncoding=utf8
      PRO_DB_USER: root
      PRO_DB_PWD: 123456
      # 若network_mode为host,则可修改端口,如下:
      # PRO_CONFIG_PORT: 8083
      # PRO_ADMIN_PORT: 8093

      # 给Portal指定远程pro地址
      #PRO_URL: http://www.example.com:8083

    ports:
      - "8070:8070"
      - "8080:8080"
      - "8081:8081"
      - "8082:8082"
      - "8083:8083"

  mysql:
    image: mysql:5.7.21
    container_name: apollo-db
    restart: always
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: 123456
    ports:
      - "13306:3306"
    volumes:
      - "./sql:/docker-entrypoint-initdb.d"
      - "./db:/var/lib/mysql"
```
