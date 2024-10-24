# 安装

## 从软件包安装
访问我们的[发布页面](https://github.com/Hornetlabs/synchdb/releases) 下载 synchdb 软件包。我们目前支持基于 Debian 的 Linux 系统（如 Ubuntu）上的 .deb 软件包。基于 CentOS 的 .rpm 软件包将在不久的将来得到支持。SynchDB .deb 软件包需要先安装 PostgreSQL。它所需的 PostgreSQL 版本在软件包名称中描述。例如，`postgresql-16-synchdb_1.0-1.22.04_amd64.deb` 是在 Ubuntu 22.04 上针对 PostgreSQL 16 构建的 .deb 包。

### .deb 包

1. 从官方 apt 存储库安装 PostgreSQL：
```sh linenums="1"
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-16
```

2. 安装 Java 运行时环境：
```sh linenums="1"
sudo apt install openjdk-17-jre-headless
```

3. 更新共享库路径：
```sh linenums="1"
JAVA_PATH=$(which java)
JRE_HOME_PATH=$(readlink -f ${JAVA_PATH} | sed 's:/bin/java::')
JRE_LIB_PATH=${JDK_HOME_PATH}/lib
echo "$JRE_LIB_PATH" | sudo tee /etc/ld.so.conf.d/java.conf
sudo ldconfig
```

4. 安装 SynchDB：
```sh linenums="1"
dpkg -i postgresql-16-synchdb_1.0-1.22.04_amd64.deb
```
5. SynchDB 应该已准备就绪。请参阅 [快速入门](https://docs.synchdb.com/user-guide/quick_start/) 页面开始使用

### .rpm 软件包
待定

## 从预编译的二进制文件安装
访问我们的[发布页面](https://github.com/Hornetlabs/synchdb/releases) 下载预编译二进制文件。我们目前支持基于 Debian 的 Linux 系统（如 Ubuntu）系统上的预编译二进制文件。其他平台将在不久的将来得到支持。SynchDB 预编译二进制文件需要先安装 PostgreSQL。它所需的 PostgreSQL 版本在包名称中描述。例如，`postgresql-16-synchdb_1.0-1.22.04_amd64.tar.gz` 是在 Ubuntu 22.04 上针对 PostgreSQL 16 编译的二进制文件。

### 预编译二进制文件
1. 解压预编译二进制文件的 tar.gz 包：
```sh linenums="1"
tar xzvf postgresql-16-synchdb_1.0-1.22.04_amd64.tar.gz -C /tmp
```

2. 找出当前 PostgreSQL 的 lib 和 share 目录
```sh linenums="1"
LIBDIR=$(pg_config | grep -w LIBDIR | awk -F ' = ' '/LIBDIR/ {print $2}')
SHAREDIR=$(pg_config | grep -w SHAREDIR | awk -F ' = ' '/SHAREDIR/ {print $2}')
```

3. 将预编译的二进制文件复制到相应目录：
```sh linenums="1"
cp /tmp/postgresql-16-synchdb_1.0-1.22.04_amd64/usr/lib/postgresql/16/lib/* $LIBDIR
cp /tmp/postgresql-16-synchdb_1.0-1.22.04_amd64/usr/share/postgresql/16/extension/* $SHAREDIR
```

4. 安装 Java Runtime Environment：
```sh linenums="1"
sudo apt install openjdk-17-jre-headless
```

5. 更新共享库路径：
```sh linenums="1"
JAVA_PATH=$(which java)
JRE_HOME_PATH=$(readlink -f ${JAVA_PATH} | sed 's:/bin/java::')
JRE_LIB_PATH=${JDK_HOME_PATH}/lib
echo "$JRE_LIB_PATH" | sudo tee /etc/ld.so.conf.d/java.conf
sudo ldconfig
```

6. SynchDB 应该已准备就绪。请参阅 [快速入门](https://docs.synchdb.com/user-guide/quick_start/) 页面开始使用

## 源代码编译及安装
此选项要求您从源代码构建和安装 PostgreSQL 和 SynchDB。

### 系统要求
构建和运行SynchDB需要以下软件。列出的版本是开发过程中测试过的版本。较旧的版本可能仍然可用。

* Java Development Kit 22. 下载地址：[点击这里](https://www.oracle.com/ca-en/java/technologies/downloads/)
* Apache Maven 3.9.8. 下载地址：[点击这里](https://maven.apache.org/download.cgi)
* PostgreSQL 16.3 源代码. Git克隆地址：[点击这里](https://github.com/postgres/postgres). PostgreSQL编译要求请参考此[wiki](https://wiki.postgresql.org/wiki/Compile_and_Install_from_source_code)
* Docker compose 2.28.1 (用于测试). 参考[安装指南](https://docs.docker.com/compose/install/linux/)
* 基于Unix的操作系统，如Ubuntu 22.04或MacOS

### 准备源代码
克隆PostgreSQL源代码并切换到16.3发布标签
```sh linenums="1"
git clone https://github.com/postgres/postgres.git
cd postgres
git checkout REL_16_3
```

在扩展文件夹中克隆SynchDB源代码
注意：目前使用*(synchdb-devel)[https://github.com/Hornetlabs/synchdb/tree/synchdb-devel]*分支进行开发。
```sh linenums="1"
cd contrib/
git clone https://github.com/Hornetlabs/synchdb.git
```

### 准备工具
#### 安装Maven
如果您使用的是Ubuntu 22.04.4 LTS，按以下方式安装Maven：
```sh
sudo apt install maven
```

如果您使用的是MacOS，可以使用brew命令安装maven（关于如何安装Homebrew，请参考(这里)[https://brew.sh/]），无需其他设置：
```sh
brew install maven
```

#### 安装Java SDK (OpenJDK)
如果您使用的是Ubuntu 22.04.4 LTS，按以下方式安装OpenJDK：
```sh
sudo apt install openjdk-21-jdk
```

如果您使用的是MacOS，请使用brew命令安装JDK：
```sh
brew install openjdk@22
```

### 编译和安装PostgreSQL
按照PostgreSQL官方文档[这里](https://www.postgresql.org/docs/current/install-make.html)从源代码编译和安装PostgreSQL。通常，步骤包括：

```sh linenums="1"
cd /home/$USER/postgres
./configure
make
sudo make install
```

您还应该编译和安装默认扩展：
```sh linenums="1"
cd /home/$USER/postgres/contrib
make
sudo make install
```

### 编译和安装Debezium Runner引擎
配置好Java和Maven后，我们就可以编译Debezium Runner引擎了。这会将Debezium Runner Engine的jar文件安装到PostgreSQL的lib文件夹中。

```sh linenums="1"
cd /home/$USER/postgres/contrib/synchdb
make build_dbz
sudo make install_dbz
```

### 编译和安装SynchDB PostgreSQL扩展
在系统中安装Java的`lib`和`include`后，可以通过以下方式编译SynchDB：

```sh linenums="1"
cd /home/$USER/postgres/contrib/synchdb
make
sudo make install
```

### 配置链接器（Ubuntu）
最后，我们还需要告诉系统链接器新添加的Java库在系统中的位置。以下步骤基于Ubuntu 22.04。

```sh linenums="1"
# 动态设置JDK路径
JAVA_PATH=$(which java)
JDK_HOME_PATH=$(readlink -f ${JAVA_PATH} | sed 's:/bin/java::')
JDK_LIB_PATH=${JDK_HOME_PATH}/lib

echo $JDK_LIB_PATH
echo $JDK_LIB_PATH/server

sudo echo "$JDK_LIB_PATH" ｜ sudo tee -a /etc/ld.so.conf.d/x86_64-linux-gnu.conf
sudo echo "$JDK_LIB_PATH/server" | sudo tee -a /etc/ld.so.conf.d/x86_64-linux-gnu.conf
```

注意：对于使用M1/M2芯片的mac，需要将这两行添加到/etc/ld.so.conf.d/aarch64-linux-gnu.conf中
```sh linenums="1"
sudo echo "$JDK_LIB_PATH"       ｜ sudo tee -a /etc/ld.so.conf.d/aarch64-linux-gnu.conf
sudo echo "$JDK_LIB_PATH/server" | sudo tee -a /etc/ld.so.conf.d/aarch64-linux-gnu.conf
```

运行ldconfig重新加载：
```sh
sudo ldconfig
```

### 检查安装

确保synchdb.so扩展可以链接到系统中的libjvm Java库：
```shsh linenums="1"
ldd synchdb.so
        linux-vdso.so.1 (0x00007ffeae35a000)
        libjvm.so => /usr/lib/jdk-22.0.1/lib/server/libjvm.so (0x00007fc1276c1000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc127498000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fc127493000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fc12748e000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007fc127489000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fc1273a0000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fc128b81000)
```