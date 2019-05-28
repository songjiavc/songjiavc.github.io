# 禅道docker安装方法
1、下载禅道官方docker版

wget http://dl.cnezsoft.com/zentao/docker/docker_zentao.zip

unzip docker_zentao.zip                             ——解压docker禅道安装包

2、cd ./docker_zentao                              ——进入docker禅道目录

docker build -t zentao ./           ————运行dockerfile生成镜像

mkdir -p /var/zentao/www 

mkdir -p /var/zentao/data           ————创建禅道文件目录数据库目录

3、运行镜像

docker run --name zentao -p 8080:80 -v /var/zentao/www:/app/zentaopms -v /var/zentao/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d zentao:latest       ————运行实例
docker update --restart=always zentao  ————设置开启服务自启

# 安装位置
docker run --name zentao -p 5001:80 -v /home/produce/dockerdata/zendao/www:/app/zentaopms -v /home/produce/dockerdata/zendao/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d zentao:latest

    
    