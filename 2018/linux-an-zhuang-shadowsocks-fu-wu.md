# Linux 安装 shadowsocks 服务

```bash
# 安装 pip
sudo yum install python-setuptools && easy_install pip
# 如果安装失败
wget https://pypi.python.org/packages/5f/ad/1fde06877a8d7d5c9b60eff7de2d452f639916a                    
unzip distribute-0.7.3.zip
cd distribute-0.7.3
sudo python setup.py install
sudo yum install python-setuptools && easy_install pip

# 安装 shadowsocks
sudo pip install shadowsocks

# 启动服务 -p 端口，默认8388 -k 密码，默认password
sudo ssserver -p 443 -k password -m aes-256-cfb --user nobody -d start
# 停止服务
# sudo ssserver -d stop
```

