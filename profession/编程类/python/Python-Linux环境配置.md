# Python Linux 环境配置
****
## 安装
&ensp;&ensp;&ensp;&ensp;按照[k-vim](https://github.com/wklken/k-vim)的git说明一路配置就行，其中要注意的是：

- pip不是应用程序，运行命令安装： sudo apt-get install python-setuptools；yum -y install python-pip

&ensp;&ensp;&ensp;&ensp;或者：
```sh
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
```

### 编辑器的安装
&ensp;&ensp;&ensp;&ensp;在远程编辑时一个可以使用的python的vim编辑时必要的，默认的vim是真的难用啊：pip install pyvim
- pip不是应用程序，运行命令安装： sudo apt-get install python-setuptools

### 切换安装第三方库源
```sh
pip3.6 install requests -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```
