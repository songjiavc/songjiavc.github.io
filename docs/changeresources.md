### 修改maven源
  修改/opt/apache-maven-3.6.1/conf/settings.xml  该配置可以针对不同的项目独立指定，避免引起不必要的冲突。  
  ex.  /home/server/.m2/settings.xml    /home/server/.m3/settings.xml  针对不同的settings.xml配置自己的localRepository 和   mirror，配置内容如下：
    
    <!-- <localRepository>/path/to/local/repo</localRepository> -->
    <localRepository>/home/songjia/.m2/repository</localRepository>  <!-- 指定本地仓库存储问题-->
     <mirror>
      <id>uk</id>
      <mirrorOf>central</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://uk.maven.org/maven2/</url>
    </mirror>
    <mirror>
      <id>nexus</id>
      <name>internal nexus repository</name>
      <url>http://repo.maven.apache.org/maven2</url>  <!--这里指定maven云仓库位置，当然也可以是你的私有仓库位置 http://192.168.15.33:8989/maven2 配置多个mirror 只有第一个生效 -->
      <mirrorOf>central</mirrorOf>
    </mirror>
### 修改docker源

修改或新增 /etc/docker/daemon.json

    vi /etc/docker/daemon.json
    {
         "registry-mirrors": ["http://hub-mirror.c.163.com"],
         "insecure-registries": [    //额外赠送，这个是在配置docker私有库时 如果不配知认证证书允许http访问时使用
            "192.168.XX.XX"
         ]
    }
systemctl restart docker.service

Docker国内源说明：

Docker 官方中国区

https://registry.docker-cn.com

中国科技大学

https://docker.mirrors.ustc.edu.cn

阿里云

https://pee6w651.mirror.aliyuncs.com


