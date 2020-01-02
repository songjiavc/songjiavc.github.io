Ubuntu 防火墙设置
sudo ufw enable
sudo ufw default deny 
作用：开启了防火墙并随系统启动同时关闭所有外部对本机的访问（本机访问外部正常）。
---
sudo ufw status 查看防火墙状态,可以查看到防火墙的白名单。