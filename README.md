# RV下的Hadoop (v3.4.0) 编译
## 前言

本教程是我在`中国科学院软件研究所`实习期间的产出，指导人为宫小飞老师。如果用此教程进行`转载 / 二创`，请标明原文链接。

## 准备依赖

Hadoop-3.4.0官方给出的依赖要求如下所示：
```bash
Requirements:

* Unix System
* JDK 1.8
* Maven 3.3 or later
* Boost 1.72 (if compiling native code)
* Protocol Buffers 3.7.1 (if compiling native code)
* CMake 3.19 or newer (if compiling native code)
* Zlib devel (if compiling native code)
* Cyrus SASL devel (if compiling native code)
* One of the compilers that support thread_local storage: GCC 9.3.0 or later, Visual Studio,
  Clang (community version), Clang (version for iOS 9 and later) (if compiling native code)
* openssl devel (if compiling native hadoop-pipes and to get the best HDFS encryption performance)
* Linux FUSE (Filesystem in Userspace) version 2.6 or above (if compiling fuse_dfs)
* Doxygen ( if compiling libhdfspp and generating the documents )
* Internet connection for first build (to fetch all Maven and Hadoop dependencies)
* python (for releasedocs)
* bats (for shell code testing)
* Node.js / bower / Ember-cli (for YARN UI v2 building)
```

部分实际用到的依赖的版本：

| 依赖              | 版本                            |
| --------------- | ----------------------------- |
| JDK             | 17                            |
| Maven           | 3.3 or later                  |
| Boost           | 1.72                          |
| Protobuf        | 2.5.0, 3.7.1, 3.21.7, 3.21.12 |
| GCC             | 12.3.1                        |
| G++             | 12.3.1                        |
| grpc-java       | 1.53.0                        |
| leveldbjni      | 1.8                           |
| leveldb-java    | 0.12                          |
| hawtjni-runtime | 1.16                          |
| yarn            | 1.22.5                        |
| node            | 12.22.1                       |

在实际编译过程中做出的部分调整：
1. JDK 1.8 ➡️ JDK 17

	JDK 17增加了对RV平台的加速，经过测试，在编译Hadoop过程中使用JDK 17 能显著加快编译速度。
	但是由于JDK 17已经移除javadoc对HTML4的支持，但是Hadoop-3.4.0中部分源码还是按照HTML4来编写，这就导致在编译过程中会报错。为适配JDK 17，需要对Hadoop-3.4.0源码进行修改，参考问题汇总中问题2。

2. Protocol Buffers 3.7.1 ➡️ Protocol Buffers 3.21.12

	由于直接编译出的Protocol Buffers 3.7.1会缺少absl库的支持，而在RV上本身又需要重新编译3.21.12，而且编译3.21.12的速度要比编译3.7.1的速度快挺多。故直接在Protobuf 3.21.12中添加编译出的abseil，并用3.21.12来编译hadoop。

### **0. yum安装部分依赖**
```bash
yum install -y gcc gcc-c++ gcc-gfortran libgcc

yum install -y wget openssl openssl-devel zlib zlib-devel automake libtool make libstdc++-static glibc-static git snappy snappy-devel fuse fuse-devel doxygen clang cyrus-sasl cyrus-sasl-devel libtirpc libtirpc-devel

yum install -y cmake maven hostname maven-local tomcat chrpath systemd leveldbjni leveldb-java hawtjni-runtime npm chrpath patch

yum install -y java-17-openjdk java-17-openjdk-devel java-17-openjdk-headless
```

### **1. 编译安装Boost 1.7.2**
Boost在安装过程中会出现failed updating *数字* targets的报错，不影响Boost编译。
```bash
# 进入仓库中的依赖目录
cd dependency

# 由于连接不稳定，有一定概率下载失败
wget https://archives.boost.io/release/1.72.0/source/boost_1_72_0.tar.gz
tar -xzf boost_1_72_0.tar.gz
pushd boost_1_72_0

./bootstrap.sh
./b2 install

popd
```

### **2. 编译安装Abseil 20240116.2**
编译安装Abseil是为编译Protobuf和grpc-java做准备，避免编译Hadoop过程中出现因为absl导致的报错。
```bash
# 为后续安装Protobuf和grpc-java做准备
wget -c https://github.com/abseil/abseil-cpp/archive/refs/tags/20240116.2.tar.gz
tar -zxf 20240116.2.tar.gz
mv abseil-cpp-20240116.2 abseil-cpp

pushd abseil-cpp

cmake ./ -DCMAKE_CXX_STANDARD=20
make
make install -j $(nproc)

popd
```

### **3. 编译安装Protobuf 2.5.0, 3.7.1, 3.21.7, 3.21.12**
Protobuf是编译Hadoop的关键。在Hadoop编译过程中会需要多种版本的Protobuf依赖，由于在RV平台上无法向在ARM平台上那样能够从远程库中拉取已经编译好的Protobuf，故每一个需要的版本都需要手动编译。

**i. 编译安装Protobuf 2.5.0**

Protobuf2.5.0的编译安装可参考下面的仓库
https://portrait.gitee.com/src-openeuler/protobuf2
```bash
# 创建Protobuf文件夹
mkdir protobuf && pushd protobuf

# 安装Protobuf 2.5.0
git clone https://gitee.com/src-openeuler/protobuf2.git
cd protobuf2
tar -xjf protobuf-2.5.0.tar.bz2
cp *.patch protobuf-2.5.0 && cd protobuf-2.5.0

# 给protobuf-2.5.0打patch以支持riscv64
patch -p1 < 0001-Add-generic-GCC-support-for-atomic-operations.patch
patch -p1 < protobuf-2.5.0-gtest.patch
patch -p1 < protobuf-2.5.0-java-fixes.patch
patch -p1 < protobuf-2.5.0-makefile.patch
patch -p1 < add-riscv64-support.patch

./configure --build=riscv64-unknown-linux --prefix=/usr/local/protobuf-2.5.0
make
make check
make install -j $(nproc)
ldconfig

# 将Protobuf2.5.0的bin文件安装到本地mvn库
mvn install:install-file -DgroupId=com.google.protobuf -DartifactId=protoc -Dversion=2.5.0 -Dclassifier=linux-riscv64 -Dpackaging=exe -Dfile=/usr/local/protobuf-2.5.0/bin/protoc

cd ../..
```

**ii. 编译安装Protobuf 3.7.1**

Protobuf3.7.1的编译过程十分缓慢。编译安装完成后，将bin文件安装到本地mvn库，以备后续使用。
```bash
# 获取Protobuf 3.7.1源码
wget -c https://github.com/protocolbuffers/protobuf/releases/download/v3.7.1/protobuf-cpp-3.7.1.tar.gz
tar -xzf protobuf-cpp-3.7.1.tar.gz

cd protobuf-3.7.1
./autogen.sh

# --prefix指定Protobuf的安装路径
./configure --prefix=/usr/local/protobuf-3.7.1
make
make check
make install -j $(nproc)
ldconfig

# 将Protobuf3.7.1的bin文件安装到本地mvn库
mvn install:install-file -DgroupId=com.google.protobuf -DartifactId=protoc -Dversion=3.7.1 -Dclassifier=linux-riscv64 -Dpackaging=exe -Dfile=/usr/local/protobuf-3.7.1/bin/protoc

cd ..
```

**iii. 编译安装Protobuf 3.21.7**

Protobuf 3.21.7 主要用于 grpc-java 1.53.0的编译。
```bash
wget -c https://github.com/protocolbuffers/protobuf/releases/download/v21.7/protobuf-cpp-3.21.7.tar.gz
tar -xzf protobuf-cpp-3.21.7.tar.gz
cd protobuf-3.21.7

# 将前面编译好的Abseil添加到Protobuf的编译中
cp -r ../abseil-cpp ./third_party

./autogen.sh

# --prefix指定Protobuf的安装路径
./configure --prefix=/usr/local/protobuf-3.21.7

make
make check
make install -j $(nproc)
ldconfig

# 将Protobuf3.21.12的bin文件安装到本地mvn库
mvn install:install-file -DgroupId=com.google.protobuf -DartifactId=protoc -Dversion=3.21.7 -Dclassifier=linux-riscv64 -Dpackaging=exe -Dfile=/usr/local/protobuf-3.21.7/bin/protoc

cd ..
```

**iiii. 编译安装Protobuf 3.21.12**

编译Protobuf 3.21.12 做为编译Hadoop时的默认Protobuf。
```bash
wget -c https://github.com/protocolbuffers/protobuf/releases/download/v21.12/protobuf-cpp-3.21.12.tar.gz
tar -xzf protobuf-cpp-3.21.12.tar.gz
cd protobuf-3.21.12

# 打补丁，以修复编译hadoop-hdfs-native-client时遇到的-fPIC问题
patch -p1 < ../../patch/protobuf/0001-add-fpic-for-hdfs-native-client.patch

cp -r ../abseil-cpp ./third_party

# 指定C++版本
cmake ./ -DCMAKE_BUILD_TYPE=RELEASE -Dprotobuf_BUILD_TESTS=off -DCMAKE_CXX_STANDARD=20

make install -j $(nproc)

# 将Protobuf3.21.12的bin文件安装到本地mvn库
mvn install:install-file -DgroupId=com.google.protobuf -DartifactId=protoc -Dversion=3.21.12 -Dclassifier=linux-riscv64 -Dpackaging=exe -Dfile=/usr/local/bin/protoc

popd
```

对Protobuf 3.21.12添加软连接，其默认安装路径为/usr/local/bin
```bash
# 软连接protoc（如已存在/usr/bin/protoc先清除后链接）
rm -rf /usr/bin/protoc
ln -s /usr/local/bin/protoc /usr/bin/protoc
```

### **4. 编译安装grpc-java-1.53.0**
```bash
mkdir -p /var/tmp/source
wget -P /var/tmp/source -c https://services.gradle.org/distributions/gradle-7.6-bin.zip

wget -c https://github.com/grpc/grpc-java/archive/refs/tags/v1.53.0.tar.gz
tar -xzf v1.53.0.tar.gz

pushd grpc-java-1.53.0

# 打补丁，添加grpc-java-1.53.0对riscv的支持
patch -p1 < ../patch/grpc/0001-add-support-for-riscv64.patch

# 将远端源换为本地源
sed -i "s,@HOME@,${HOME},g" build.gradle
sed -i 's|https\\://services.gradle.org/distributions|file:///var/tmp/source|g' gradle/wrapper/gradle-wrapper.properties

# 指定架构为riscv64
SKIP_TESTS=true ARCH=riscv64 ./buildscripts/kokoro/unix.sh

# 将 grpc-1.53.0的bin文件安装到本地mvn库
mvn install:install-file -DgroupId=io.grpc -DartifactId=protoc-gen-grpc-java -Dversion=1.53.0 -Dclassifier=linux-riscv64 -Dpackaging=exe -Dfile=mvn-artifacts/io/grpc/protoc-gen-grpc-java/1.53.0/protoc-gen-grpc-java-1.53.0-linux-riscv64.exe
popd

```

### **5. 安装leveldb相关依赖**
由于hadoop编译过程中需要用到mvn库中一些leveldb的相关依赖，但是远程源中并没有对应riscv版本，故无法直接远程拉取，需要在本地通过yum安装过后，手动添加到本地mvn库。
要根据实际安装的版本设置参数。
```bash
yum install -y \
	leveldbjni \
	leveldb-java \
	hawtjni-runtime

# leveldbjni-all.jar
mvn install:install-file -DgroupId=org.fusesource.leveldbjni -DartifactId=leveldbjni-all -Dversion=1.8 -Dpackaging=jar -Dfile=/usr/lib/java/leveldbjni-all.jar

# leveldbjni.jar
mvn install:install-file -DgroupId=org.fusesource.leveldbjni -DartifactId=leveldbjni -Dversion=1.8 -Dpackaging=jar -Dfile=/usr/lib/java/leveldbjni/leveldbjni.jar

# leveldb-api.jar
mvn install:install-file -DgroupId=org.iq80.leveldb -DartifactId=leveldb-api -Dversion=0.12 -Dpackaging=jar -Dfile=/usr/share/java/leveldb-java/leveldb-api.jar

# leveldb-benchmark.jar
mvn install:install-file -DgroupId=org.iq80.leveldb -DartifactId=leveldb-benchmark -Dversion=0.12 -Dpackaging=jar -Dfile=/usr/share/java/leveldb-java/leveldb-benchmark.jar

# leveldb.jar
mvn install:install-file -DgroupId=org.iq80.leveldb -DartifactId=leveldb -Dversion=0.12 -Dpackaging=jar -Dfile=/usr/share/java/leveldb-java/leveldb.jar

# hawtjni-runtime.jar
mvn install:install-file -DgroupId=orn.fusesource.hawtjni -DartifactId=hawtjni-runtime -Dversion=1.16 -Dpackaging=jar -Dfile=/usr/lib/java/hawtjni/hawtjni-runtime.jar
```

### **6. 安装yarn-1.22.5和node-12.22.1**
同样由于无法从远程源直接获取，故手动安装。
```bash
# 安装node-12.22.1
mkdir -p ${HOME}/.m2/repository/com/github/eirslett/node/12.22.1/

# 由于未找到12.22.1版本的riscv版，故选择用14.8.0欺骗一下
cp node-v14.8.0-linux-riscv64.tar.xz ${HOME}/.m2/repository/com/github/eirslett/node/12.22.1/
pushd ${HOME}/.m2/repository/com/github/eirslett/node/12.22.1/
tar -xJf node-v14.8.0-linux-riscv64.tar.xz

# node-v14.8.0重命名为node-v12.22.1
mv node-v14.8.0-linux-riscv64 node-v12.22.1-linux-x64

tar -zcf node-12.22.1-linux-x64.tar.gz node-v12.22.1-linux-x64
rm -rf node-v12.22.1-linux-x64
popd

# 安装yarn-1.22.5
mkdir -p ${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/

cp yarn-v1.22.5.tar.gz ${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/

mv ${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/yarn-v1.22.5.tar.gz ${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/yarn-1.22.5.tar.gz

tar -xzvf ${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/yarn-1.22.5.tar.gz -C ${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/

# 修改npm镜像源
npm config set registry https://repo.huaweicloud.com/repository/npm/

# 清理npm缓存
npm cache clean -f

# 修改yarn镜像源
${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/yarn-v1.22.5/bin/yarn config set registry https://repo.huaweicloud.com/repository/npm/ -g

# 设置yarn引擎
${HOME}/.m2/repository/com/github/eirslett/yarn/1.22.5/yarn-v1.22.5/bin/yarn config set ignore-engines true

```

### **7. 添加绑定native库相关依赖**

**i. 编译安装zstd 1.5.6**
```bash
git clone https://github.com/facebook/zstd.git -b v1.5.6
cd zstd

make
make install

cd ..
```

**ii. 编译安装isa-l 2.31.0**
```bash
git clone https://github.com/intel/isa-l.git -b v2.31.0
cd isa-l

# 安装所需依赖
yum install -y yasm
yum install -y nasm
yum install -y help2man

./autogen.sh
./configure --prefix=/usr --libdir=/usr/lib64

make
make install

cd ..
```

**iii. 编译安装kmod 28 (为pmdk做准备)**
```bash
git clone https://github.com/lucasdemarchi/kmod.git -b v28
cd kmod

# 安装所需依赖
yum install -y gtk-doc

./autogen.sh
./configure CFLAGS="-g -O2" --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib64

make
make install

cd ..
```

**iiii. 以直装方式准备daxctl和ndctl (为pmdk做准备)**
```bash
yum install -y asciidoc asciidoctor xmlto

yum install -y systemd-devel uuid-devel json-c-devel iniparser

yum install -y bash-completion

yum install -y keyutils keyutils-libs keyutils-libs-devel

yum install -y daxctl-libs daxctl-devel daxctl ndctl-libs ndctl-devel ndctl
```

**v. 安装pandoc 3.4 (为pmdk做准备)**
```bash
# 使用别人编译好的riscv版本pandoc
wget -c https://dl.b-data.ch/pandoc/3.4/pandoc-3.4-linux-riscv64.tar.gz

tar -zxf pandoc-3.4-linux-riscv64.tar.gz
```
构建pandoc的软链接
```bash
rm -rf /usr/bin/pandoc

# 根据pandoc解压位置设置
pandoc_dir=$(pwd)
ln -s $pandoc_dir/pandoc-3.4/bin/pandoc /usr/bin/pandoc
```

**vi. 编译安装pmdk 2.1.0**

在软件所提供的几台RV机器上如果gcc的版本为12.3.0，编译时会报错；可以使用yum中的12.3.1版本。
```bash
git clone https://github.com/pmem/pmdk.git -b 2.1.0
cd pmdk

make -j
make install

cd ..
```

## 编译Hadoop

### **0. 设置编译环境**
添加Maven使用资源的限制防止崩溃，设置使用JDK17
```bash
export MAVEN_OPTS="-Xms2048M -Xmx8000M"

#JDK17路径，根据实际设置
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-17.0.10.7-2.oe2403.riscv64
export PATH=$PATH:$JAVA_HOME/bin
```

### **1. 下载Hadoop 3.4.0源码**
```bash
# 回到仓库主目录
cd ..
git clone https://github.com/apache/hadoop.git -b rel/release-3.4.0

```

### **2. 打补丁以支持在RV环境中的编译**
```bash
cp patch/hadoop/*.patch hadoop
pushd hadoop

# 添加triple-beam-1.3.0
patch -p1 < 0001-lock-triple-beam-version-to-1.3.0.patch

# 修复mvn无法识别riscv架构的问题
patch -p1 < 0002-upgrade-maven-plugins.patch

# 在hadoop-mapreduce-client-nativetask中添加对riscv64的支持
patch -p1 < 0003-add-riscv-bswap-function.patch

# 修复pom文件以指定leveldb相关依赖在本地的位置
patch -p1 < 0004-fix-pom.patch

# 修复hadoop-3.4.0对JDK 17的支持
patch -p1 < 0005-make-javadoc-work-on-java-17.patch

# 修复编译时openssl的native库问题
patch -p1 < 0006-fix-openssl-native.patch
```

### **3. 开始编译**
补丁打完后就可以开始Hadoop的正式编译，核心命令为：
```bash
mvn package -T20 -Pdist,native -DskipTests -Dtar -Dmaven.compiler.compilerArgs=-O0 -DargLine=-Xint -Dfailsafe.argLine=-Xint -Drequire.snappy -Drequire.zstd -Drequire.openssl -Drequire.isal -Drequire.pmdk -e
```

涉及参数说明：

| 命令/参数                             | 说明                          |
| --------------------------------- | --------------------------- |
| package                           | 创建JAR                       |
| -Pdist,native                     | 使用源码构建二进制发行版                |
| -DskipTests                       | 在编译过程中让Maven跳过test环节        |
| -Dtar                             | 将编译结果整合为tar包                |
| -Dmaven.compiler.compilerArgs=-O0 | 在编译过程中让Maven关闭编译优化，用于”控制变量“ |
| -DargLine=-Xint                   | 让单元测试在解释环境下进行，避免优化，用于”控制变量“ |
| -Dfailsafe.argLine=-Xint          | 让集成测试在解释环境下进行，避免优化，用于”控制变量“ |
| -Drequire.snappy                  | 在编译结果中添加对snappy的使用          |
| -Drequire.zstd                    | 在编译结果中添加对zstd的使用            |
| -Drequire.openssl                 | 在编译结果中添加对openssl的使用         |
| -Drequire.isal                    | 在编译结果中添加对isal的使用            |
| -Drequire.pmdk                    | 在编译结果中添加对pmdk的使用            |
| -e                                | 如果遇到编译错误，输出错误日志             |

## 问题汇总
这是我在编译中增经遇到的一些问题及解决办法，可供参考：

#### 1. 编译hadoop过程中遇到org.fusesource.leveldbjni.internal和org.fusesource.leveldbjni.internal缺失的问题

解决办法：先将缺失的包打好patch后编译安装或通过yum安装，再通过mvn install部署到本地repository中。并同时在相应project的pom.xml文件中正确指定依赖（参考0004-fix-pom.patch）。

#### 2. 编译hadoop时hadoop-auth模块报错
```bash
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-javadoc-plugin:3.0.1:jar (module-javadocs) on project hadoop-maven-plugins: MavenReportException: Error while generating

 Javadoc:

[ERROR] Exit code: 1 - error: invalid flag: -html4

[ERROR] 1 error
```
**解决办法**：新版jdk已移除javadoc对HTML4的支持，需要修改源码以支持html5。将github上关于修改源码以使javadoc可以在jdk17下工作的PR生成patch（0005-make-javadoc-work-on-java-17.patch），并打入源码中。参考链接如下：
[https://github.com/apache/hadoop/pull/6976](https://github.com/apache/hadoop/pull/6976)

#### 3. 编译hadoop时hadoop-hdfs-native-client模块报错
```bash
[WARNING] /usr/local/include/absl/types/compare.h:455:56: error: ‘weak_ordering’ in namespace ‘absl’ does not name a type

[WARNING]   455 | constexpr bool compare_result_as_less_than(const absl::weak_ordering r) {

[WARNING]       |                                                        ^~~~~~~~~~~~~

[WARNING] /usr/local/include/absl/types/compare.h:470:17: error: ‘weak_ordering’ in namespace ‘absl’ does not name a type

[WARNING]   470 | constexpr absl::weak_ordering compare_result_as_ordering(const Int c) {

[WARNING]       |                 ^~~~~~~~~~~~~

[WARNING] /usr/local/include/absl/types/compare.h:475:17: error: ‘weak_ordering’ in namespace ‘absl’ does not name a type

[WARNING]   475 | constexpr absl::weak_ordering compare_result_as_ordering(

[WARNING]       |                 ^~~~~~~~~~~~~
```
**解决办法**：重新编译abseil，并在编译protobuf时，将编译出的abseil添加到third_party文件夹中，最后重新编译grpc-java（可参考上面的构建脚本）。

#### 4. 编译hadoop时hadoop-hdfs-native-client模块报错
```bash
[WARNING] /usr/bin/ld: /usr/local/lib64/libprotobuf.a(arena.cc.o): relocation R_RISCV_HI20 against `a local symbol' can not be used when making a shared object; recompile with -fPIC             ^~~~~~~~~~~~~
```
**解决办法**：这是链接 libprotobuf.a 时的重定位问题，在编译动态库时没有使用-fPIC选项，需要对protobuf重新编译。为了生成位置无关代码，在protobuf源码的CMakeLists.txt中加入
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
详见0001-add-fpic-for-hdfs-native-client.patch

#### 5. 编译hadoop时hadoop-yarn-applications-catalog-webapp模块报错
```bash
[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.11.2:yarn (yarn install) on project hadoop-yarn-applications-catalog-webapp: Failed to run task: 'yarn ' failed. org.apache.commons.exec.ExecuteException: Process exited with an error: 126 (Exit value: 126) -> [Help 1] org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.11.2:yarn (yarn install) on project hadoop-yarn-applications-catalog-webapp: Failed to run task [INFO] Running 'yarn ' in /root/hadoop/hadoop/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog/hadoop-yarn-applications-catalog-webapp/target [INFO] /root/hadoop/hadoop/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog/hadoop-yarn-applications-catalog-webapp/target/node/yarn/dist/bin/yarn: line 20: /root/hadoop/hadoop/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog/hadoop-yarn-applications-catalog-webapp/target/node/node: cannot execute binary file: Exec format error
```
**解决办法**：问题出自x86的node-v12.22.1无法在RV上运行。使用rv架构的node-v14.8.0-linux-riscv64，并修改文件名来适配要求。
[https://github.com/v8-riscv/node/releases/download/v14.8.0-riscv64/node-v14.8.0-linux-riscv64.tar.xz](https://github.com/v8-riscv/node/releases/download/v14.8.0-riscv64/node-v14.8.0-linux-riscv64.tar.xz)

#### 6. 编译hadoop时hadoop-yarn-applications-catalog-webapp模块报错
```bash
[ERROR] Failed to execute goal com.github.searls:jasmine-maven-plugin:2.1:test (default) on project hadoop-yarn-applications-catalog-webapp: The jasmine-maven-plugin encountered an exception: [ERROR] org.openqa.selenium.remote.UnreachableBrowserException: Could not start a new session. Possible causes are invalid address of the remote server or browser start-up failure. [ERROR] Build info: version: '2.48.2', revision: '41bccdd10cf2c0560f637404c2d96164b67d9d67', time: '2015-10-09 13:08:06' [ERROR] System info: host: 'openeuler-riscv-4-3', ip: '192.168.102.127', os.name: 'Linux', os.arch: 'riscv64', os.version: '6.6.0-39.0.0.47.eos30.riscv64', java.version: '17.0.10' [ERROR] Driver info: driver.version: PhantomJSDriver
```
**解决办法**：问题应该是出自jasmine-maven-plugin 2.1与jdk17不匹配，无法用jdk17编译，但是用jdk8去编译时会卡在CycloneDX: Resolving Dependencies。通过在hadoop-project/pom.xml中将jasmine-maven-plugin的版本升级为2.2，再用jdk17编译即可。

#### 7. 验证hadoop的native库编译情况时openssl, ISA-L, PMDK部分缺失
```bash
openssl: false EVP_CIPHER_CTX_block_size

ISA-L: false libhadoop was built without ISA-L support

PMDK: false The native code was built without PMDK support
```
**解决办法**：ISA-L和PMDK需要重新编译，编译步骤可参考 构建流程 (14)。openssl可用过yum安装，但在openssl 3.x中部分变量名发生了更改，例如：
EVP_CIPHER_CTX_block_size → EVP_CIPHER_CTX_get_block_size
需要打补丁修复，见0006-fix-openssl-native.patch
[https://issues.apache.org/jira/browse/HADOOP-18583](https://issues.apache.org/jira/browse/HADOOP-18583)

#### 8. 编译 PMDK报错
```bash
LD_LIBRARY_PATH=:/usr/lib64:/usr/local/lib cc -o pminvaders pminvaders.o -Wl,-rpath=../../../debug -L../../../debug -lpmemobj -lpmem -pthread -lncurses -ltinfo   

/usr/bin/ld: unresolvable R_RISCV_TPREL_HI20 relocation against symbol `_pobj_cached_pool@@LIBPMEMOBJ_1.0'

collect2: error: ld returned 1 exit status"""
```
**解决办法**：通过不同服务器对比发现，问题只出现在gcc-12.3.0情况下，gcc-12.3.1情况下可以正常编译。
