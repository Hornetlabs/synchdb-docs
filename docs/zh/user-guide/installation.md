# 安装

## 要求
构建和运行 SynchDB 需要以下软件。列出的版本是开发过程中测试的版本。旧版本可能仍然可以使用。
* Java 开发工具包 22。下载地址 [点击这里](https://www.oracle.com/ca-en/java/technologies/downloads/)
* Apache Maven 3.9.8。下载地址 [点击这里](https://maven.apache.org/download.cgi)
* PostgreSQL 16.3 源代码。Git 克隆地址 [点击这里](https://github.com/postgres/postgres)。参考这个 [wiki](https://wiki.postgresql.org/wiki/Compile_and_Install_from_source_code) 了解 PostgreSQL 构建要求
* Docker compose 2.28.1（用于测试）。参考 [这里](https://docs.docker.com/compose/install/linux/)
* 基于 Unix 的操作系统，如 Ubuntu 22.04 或 MacOS

## 准备源代码
克隆 PostgreSQL 源代码并切换到 16.3 
```
git clone https://github.com/postgres/postgres.git
cd postgres
git checkout REL_16_3
```

在扩展文件夹内克隆 SynchDB 源代码
注意：目前使用的是 *(synchdb-devel)[https://github.com/Hornetlabs/synchdb/tree/synchdb-devel]* 分支进行开发。

```
cd contrib/
git clone https://github.com/Hornetlabs/synchdb.git
```

## 准备工具
### 安装 Maven
如果你使用的是 Ubuntu 22.04.4 LTS，按以下方式安装 Maven：
```
sudo apt install maven
```

如果你使用的是 MacOS，你可以使用 brew 命令安装 Maven（参考 (这里)[https://brew.sh/] 了解如何安装 Homebrew），无需其他设置：
```
brew install maven
```

### 安装 Java SDK (OpenJDK)
如果你使用的是 Ubuntu 22.04.4 LTS，按以下方式安装 OpenJDK：
```
sudo apt install openjdk-21-jdk
```

如果你使用的是 MacOS，请使用 brew 命令安装 JDK：
```
brew install openjdk@22
```
## 构建和安装 PostgreSQL
按照官方 PostgreSQL 文档 [这里](https://www.postgresql.org/docs/current/install-make.html) 的说明从源代码构建和安装 PostgreSQL。通常，步骤包括：

```
cd /home/$USER/postgres
./configure
make
sudo make install
```

你还应该构建和安装默认扩展：
```
cd /home/$USER/postgres/contrib
make
sudo make install
```

## 构建和安装 Debezium Runner 引擎
设置好 Java 和 Maven 后，我们就可以构建 Debezium Runner 引擎了。这会将 Debezium Runner 引擎的 jar 文件安装到你的 PostgreSQL 的 lib 文件夹中。

```
cd /home/$USER/postgres/contrib/synchdb
make build_dbz
sudo make install_dbz
```

## 构建和安装 SynchDB PostgreSQL 扩展
在系统中安装了 Java 的 `lib` 和 `include` 后，可以通过以下方式构建 SynchDB：

```
cd /home/$USER/postgres/contrib/synchdb
make
sudo make install
```

## 配置你的链接器（Ubuntu）
最后，我们还需要告诉系统的链接器新添加的 Java 库在系统中的位置。以下步骤基于 Ubuntu 22.04。

```
# 动态设置 JDK 路径
JAVA_PATH=$(which java)
JDK_HOME_PATH=$(readlink -f ${JAVA_PATH} | sed 's:/bin/java::')
JDK_LIB_PATH=${JDK_HOME_PATH}/lib

echo $JDK_LIB_PATH
echo $JDK_LIB_PATH/server

sudo echo "$JDK_LIB_PATH" ｜ sudo tee -a /etc/ld.so.conf.d/x86_64-linux-gnu.conf
sudo echo "$JDK_LIB_PATH/server" | sudo tee -a /etc/ld.so.conf.d/x86_64-linux-gnu.conf
```
注意，对于使用 M1/M2 芯片的 Mac，你需要将这两行添加到 /etc/ld.so.conf.d/aarch64-linux-gnu.conf 中

运行 ldconfig 重新加载：
```
sudo ldconfig
```
## 检查安装

确保 synchdb.so 扩展可以链接到系统上的 libjvm Java 库：
```
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