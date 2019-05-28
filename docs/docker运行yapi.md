### 内容简介
高效、易用、功能强大的API管理平台
旨在为开发、产品、测试人员提供更优雅的接口管理服务
### docker 安装参考
    https://hub.docker.com/r/crper/yapi/
    docker volume create yapi-mongo    //创建一个储存卷,用来专门存放yapi使用的mongodb的数据
    docker run -d --name yapi-mongo -v yapi-mongo:/data/db mongo
    
### 访问路径和帐号
    yapi 接口管理系统http://192.168.0.206:3000/ 管理员账号crper@outlook.com 密码 ymfe.org  
