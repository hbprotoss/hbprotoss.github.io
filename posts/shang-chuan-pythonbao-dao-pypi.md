<!-- 
.. title: 上传python包到pypi
.. slug: shang-chuan-pythonbao-dao-pypi
.. date: 2017-09-30 15:51:05 UTC+08:00
.. tags: Python, PyPi
.. category: 
.. link: 
.. description: 
.. type: text
-->

## 注册账号

* 正式仓库: [PyPI Live](https://pypi.python.org/pypi?%3Aaction=register_form)
* 测试仓库: [PyPI Test](https://testpypi.python.org/pypi?%3Aaction=register_form)


## 创建.pypirc文件

```
[distutils]
index-servers =
  pypi
  pypitest

[pypi]
repository=https://upload.pypi.org/legacy/
username=your_username
password=your_password

[pypitest]
repository=https://test.pypi.org/legacy/
username=your_username
password=your_password
```

保存到`~/.pypirc`

```
chmod 600 ~/.pypirc
```

## 准备工作

工作目录结构

```
root-dir/   # arbitrary working directory name
  setup.py
  setup.cfg
  LICENSE.txt
  README.md
  protoss-pypi/
    __init__.py
    foo.py
    bar.py
    baz.py
```

### setup.py

```py
from distutils.core import setup
setup(
    name='protoss-pypi',
    packages=['protoss-pypi'],
    version='0.1.1',
    description='A random test lib',
    author='hbprotoss',
    author_email='gamespy1991@gmail.com',
    url='https://github.com/hbprotoss/pypitest',
    download_url='https://github.com/hbprotoss/pypitest/archive/master.zip',
    keywords=['testing', 'logging', 'example'],  # arbitrary keywords
    classifiers=[],
)

```

name和packages保持一致，url写git仓库地址，download_url写源码包地址

### setup.cfg

markdown写的readme文件需要显示指定

```
[metadata]
description-file = README.md
```

## 打包上传

### 正式

```shell
python setup.py sdist upload -r pypi
```

### 测试

```shell
python setup.py sdist upload -r pypitest
```