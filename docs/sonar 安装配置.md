### sonar 配置
    目前sonar 容器可以直连mysql 容器，未来我们所有的组建都容器化，除了k8s 没有非容器化的组建，所有内容都运行在容器中。
    
安装中生成了tokencode 25f5d9df5a7ac5eaadb024809ec8e1e6205220bb
安装成功后 可以通过在pom.xml中写入插件

    mvn sonar:sonar \
     -Dsonar.host.url=http://192.168.0.208:9000 \
     -Dsonar.login=25f5d9df5a7ac5eaadb024809ec8e1e6205220bb