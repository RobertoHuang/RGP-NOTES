# `JDK`安装

- 查看并卸载`CentOS`自带的`OpenJDK`

  > 安装好的`CentOS`会自带`OpenJdk`，用命令`java -version`会有下面的信息
  >
  > ```reStructuredText
  > [root@localhost ~]# java -version
  > java version "1.6.0"
  > OpenJDK Runtime Environment (build 1.6.0-b09)
  > OpenJDK 64-Bit Server VM (build 1.6.0-b09, mixed mode)
  > ```

  最好还是先卸载掉`openjdk`再安装`sun`公司的`JDK`

  - `rpm -qa | grep java` 或`rpm -qa | grep jdk`命令来查询出系统自带的`JDK`，显示如下信息:

    ```reStructuredText
    java-1.4.2-gcj-compat-1.4.2.0-40jpp.115
    java-1.6.0-openjdk-1.6.0.0-1.7.b09.el5
    ```

  - `rpm -e --nodeps`卸载系统自带`JDK`

    ```reStructuredText
    rpm -e --nodeps java-1.4.2-gcj-compat-1.4.2.0-40jpp.115
    rpm -e --nodeps java-1.6.0-openjdk-1.6.0.0-1.7.b09.el5
    
    如果出现找不到openjdk source的话，那么还可以这样卸载
    yum -y remove java java-1.4.2-gcj-compat-1.4.2.0-40jpp.115
    yum -y remove java java-1.6.0-openjdk-1.6.0.0-1.7.b09.el5
    ```

- 下载`JDK`解压到指定安装目录(略)

- 修改`etc/profile`添加如下配置，使用`source /etc/profile`使配置生效

  ```reStructuredText
  export JAVA_HOME=/opt/java/jdk1.8.0_91
  export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
  export PATH=$PATH:$JAVA_HOME/bin
  ```

  请记住在上述添加过程中等号两侧不要加入空格，不然会出现"不是有效的标识符"