# 安装环境
    ubuntu 16.04 server 64bit
    maven 3.1
    jdk 1.8.17
    docker 18.06.1-ce
# docker 安装与配置
  _安装参考:_  https://docs.docker.com/install/linux/docker-ce/ubuntu  
  _下载地址:_  https://download.docker.com/linux/ubuntu/dists/xenial/pool/stable/amd64/docker-ce_18.06.3~ce~3-0~ubuntu_amd64.deb
## docker 安装
    $ sudo dpkg -i /path/to/package.deb
## docker切换源地址
	vi /etc/docker/daemon.json
    {
  		"registry-mirrors": ["https://registry.docker-cn.com"]
	}
## 创建docker分组
	$ sudo groupadd docker
    $ sudo usermod -aG docker $USER
    这样我们运行docker命令就不用加sudo
    docker version 验证docker安装是否成功和docker版本。
# gitlab 安装与配置
_参考:_ https://docs.gitlab.com/omnibus/docker/README.html 
里面写的很详细，不过如果不能阅读英文文档还是要先攻克一下英语。
镜像下载完成执行镜像命令
sudo docker run -d \      //后台运行
  --publish 443:443       //https 端口映射
  --publish 5003:80 \	  //web应用访问端口映射
  --publish 220:22 \      //ssh 端口映射
  --name gitlab \
  --restart always \
  --volume /home/produce/dockerdata/gitlab/config:/etc/gitlab \
  --volume /home/produce/dockerdata/gitlab/logs:/var/log/gitlab \
  --volume /home/produce/dockerdata/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest
  配置/etc/gitlab/gitlab.rb
 external_url "http://gitlab.example.com"   //不能加端口号 否则无法访问
### 建立jenkins账号，为了在jenkins中下载gitlab上的代码使用
![1](../images/1.png)
*注意： 创建jenkins账号最好建立Admin权限的，这样你可以git clone所有的gitlab上的项目，比较方便*

![2](../images/2.png)

如图所示，在你要进行持续集成的项目中，选中Integrations选项，建立webhook，该webhook的地址是你 jenkins 创建project里展示的地址，如下图所示






 
 
# jenkins安装与配置