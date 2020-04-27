# 0 编译过程中遇到的问题

## 1.go依赖无法下载的问题

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.cn	
```

关于go的一些依赖方面的知识

[go module](https://colobu.com/2018/08/27/learn-go-module/)

[go依赖管理工具](https://segmentfault.com/a/1190000020543746)

## 2.make unit-test 失败问题

[官方文档：unit-test](https://hyperledger-fabric.readthedocs.io/en/release-2.0/dev-setup/build.html#running-the-unit-tests)

1. 安装libssl-dev

   sudo apt install libssl-dev -y

2. 需要安装softhsm [安装介绍](https://www.howtoforge.com/tutorial/how-to-install-and-use-softhsm-on-ubuntu-1604-lts/)



ubutu: 

```bash
sudo apt install libsofthsm2 

cd $HOME
mkdir -p $HOME/lib/softhsm/tokens
cd $HOME/lib/softhsm/
echo "directories.tokendir = $PWD/tokens" > softhsm2.conf
# 建议写在环境变量里
export SOFTHSM2_CONF=$HOME/lib/softhsm/softhsm2.conf

softhsm2-util --init-token --slot 0 --label "ForFabric" --so-pin 1234 --pin 98765432

export PKCS11_LIB="/usr/local/lib/softhsm/libsofthsm2.so"
export PKCS11_PIN=98765432
export PKCS11_LABEL="ForFabric"
```



## 3.make integration-test 失败问题