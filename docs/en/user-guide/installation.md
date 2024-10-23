# Installation

## Requirements
The following software is required to build and run SynchDB. The versions listed are the versions tested during development. Older versions may still work.

* Java Development Kit 22. Download [here](https://www.oracle.com/ca-en/java/technologies/downloads/)
* Apache Maven 3.9.8. Download [here](https://maven.apache.org/download.cgi)
* PostgreSQL 16.3 Source. Git clone [here](https://github.com/postgres/postgres). Refer to this [wiki](https://wiki.postgresql.org/wiki/Compile_and_Install_from_source_code) for PostgreSQL build requirements
* Docker compose 2.28.1 (for testing). Refer to [here](https://docs.docker.com/compose/install/linux/)
* Unix based operating system like Ubuntu 22.04 or MacOS

## Prepare Source
Clone the PostgreSQL source and switch to 16.3 release tag
```sh linenums="1"
git clone https://github.com/postgres/postgres.git
cd postgres
git checkout REL_16_3
```

Clone the SynchDB source from within the extension folder
Note: Branch *(synchdb-devel)[https://github.com/Hornetlabs/synchdb/tree/synchdb-devel]* is used for development so far.
```sh linenums="1"
cd contrib/
git clone https://github.com/Hornetlabs/synchdb.git
```

## Prepare Tools
### Install Maven
If you are working on Ubuntu 22.04.4 LTS, install the Maven as below:
```sh
sudo apt install maven
```

if you are using MacOS, you can use the brew command to install maven (refer (here)[https://brew.sh/] for how to install Homebrew) without any other settings:
```sh
brew install maven
```

### Install Java SDK (OpenJDK)
If you are working on Ubuntu 22.04.4 LTS, install the OpenJDK  as below:
```sh
sudo apt install openjdk-21-jdk
```

If you are working on MacOS, please install the JDK with brew command:
```sh
brew install openjdk@22
```

## Build and Install PostgreSQL
Follow the official PostgreSQL documentation [here](https://www.postgresql.org/docs/current/install-make.html) to build and install PostgreSQL from source. Generally, the procedure consists of:

```sh linenums="1"
cd /home/$USER/postgres
./configure
make
sudo make install
```

You should build and install the default extensions as well:
```sh linenums="1"
cd /home/$USER/postgres/contrib
make
sudo make install
```

## Build and Install Debezium Runner Engine
With Java and Maven setup, we are ready to build Debezium Runner Engine. This installs the Debezium Runner Engine jar file to your PostgreSQL's lib folder.

```sh linenums="1"
cd /home/$USER/postgres/contrib/synchdb
make build_dbz
sudo make install_dbz
```

## Build and Install SynchDB PostgreSQL Extension
With the Java `lib` and `include` installed in your system, SynchDB can be built by:

```sh linenums="1"
cd /home/$USER/postgres/contrib/synchdb
make
sudo make install
```

## Configure Your Linker (Ubuntu)
Lastly, we also need to tell your system's linker where the newly added Java library is located in your system. The following procedure is based on Ubuntu 22.04.

```sh linenums="1"
# Dynamically set JDK paths
JAVA_PATH=$(which java)
JDK_HOME_PATH=$(readlink -f ${JAVA_PATH} | sed 's:/bin/java::')
JDK_LIB_PATH=${JDK_HOME_PATH}/lib

echo $JDK_LIB_PATH
echo $JDK_LIB_PATH/server

sudo echo "$JDK_LIB_PATH" ｜ sudo tee -a /etc/ld.so.conf.d/x86_64-linux-gnu.conf
sudo echo "$JDK_LIB_PATH/server" | sudo tee -a /etc/ld.so.conf.d/x86_64-linux-gnu.conf

```

Note, for mac with M1/M2 chips, you need to the two lines into /etc/ld.so.conf.d/aarch64-linux-gnu.conf

```sh linenums="1"
sudo echo "$JDK_LIB_PATH"       ｜ sudo tee -a /etc/ld.so.conf.d/aarch64-linux-gnu.conf
sudo echo "$JDK_LIB_PATH/server" | sudo tee -a /etc/ld.so.conf.d/aarch64-linux-gnu.conf
```

Run ldconfig to reload:
```sh
sudo ldconfig
```
## Check Installation

Ensure synchdo.so extension can link to libjvm Java library on your system:
```sh linenums="1"
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
