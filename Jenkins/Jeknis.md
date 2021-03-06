# Jenkins搭建全记录
## 安装Jenkins
安装请参见文档:
 <a href="https://jenkins.io/doc/pipeline/tour/getting-started/">jenkins.io</a>
## 服务配置
我们期望的是将jenkins作为一个服务，可以开机自动启动，支持start、stop、restart、status命令。
 
### 启动脚本
参见注释配置基本的环境变量  
并将启动脚本放置在JENKINS_HOME/start-jenkins.sh
chmod a+x start-jenkins.sh
 ```bash
#!/bin/bash

#init
#Jenkins安装目录，即jenkins.war所在的目录
JENKINS_HOME=/home/service/jenkins
#jenkins输出的log文件
JENKINS_LOG=${JENKINS_HOME}/logs/jenkins.out
#java命令的位置
JENKINS_JAVA=/home/jdk/jdk1.8.0_201/bin/java
#虚拟机启动参数
JENKINS_JAVAOPTS=""
#jenkins监听的http ip
JENKINS_IP=""
#http port
JENKINS_PORT""
#参数
JENKINS_ARGS=""
# import sysconfig settings and set defaults
[ -f /etc/sysconfig/jenkins ] && . /etc/sysconfig/jenkins
[ "${JENKINS_HOME}" == "" ] &&
    JENKINS_HOME=/usr/local/jenkins
[ "${JENKINS_LOG}" == "" ] &&
    JENKINS_LOG=/home/jenkins/jenkins.log
[ "${JENKINS_JAVA}" == "" ] &&
    JENKINS_JAVA=/usr/bin/java
[ "${JENKINS_JAVAOPTS}" == "" ] &&
    JENKINS_JAVAOPTS=""
[ "${JENKINS_IP}" == "" ] &&
    JENKINS_IP=0.0.0.0
[ "${JENKINS_PORT}" == "" ] &&
    JENKINS_PORT=8080
[ "${JENKINS_ARGS}" == "" ] &&
    JENKINS_ARGS=""

JENKINS_WAR=${JENKINS_HOME}/jenkins.war

# check for config errors
JENKINS_ERRORS=()
[ ! -f ${JENKINS_WAR} ] &&
    JENKINS_ERRORS[${#JENKINS_ERRORS[*]}]="JENKINS_HOME : The jenkins.war could not be found at ${JENKINS_HOME}/jenkins.war"
[ ! -f $JENKINS_JAVA ] &&
    JENKINS_ERRORS[${#JENKINS_ERRORS[*]}]="JENKINS_JAVA : The java executable could not be found at $JENKINS_JAVA"

# display errors if there are any, otherwise start the process
if [ ${#JENKINS_ERRORS[*]} != '0' ]
then
    echo "CONFIGURATION ERROR:"
    echo "    The following errors occurred when starting Jenkins."
    echo "    Please set the appropriate values at /etc/sysconfig/jenkins"
    echo ""
    for (( i=0; i<${#JENKINS_ERRORS[*]}; i++ ))
    do
        echo "${JENKINS_ERRORS[${i}]}"
    done
    echo ""
    exit 1
else
    echo "starting service"
    echo "nohup nice $JENKINS_JAVA $JENKINS_JAVAOPTS -jar $JENKINS_WAR --httpListenAddress=$JENKINS_IP --httpPort=$JENKINS_PORT $> $JENKINS_LOG 2>&1 &"
    nohup nice $JENKINS_JAVA $JENKINS_JAVAOPTS -jar $JENKINS_WAR --httpListenAddress=$JENKINS_IP --httpPort=$JENKINS_PORT $> $JENKINS_LOG 2>&1 &
fi
```
### 停止脚本
将启动脚本放置在JENKINS_HOME/stop-jenkins.sh
chmod a+x stop-jenkins.sh
```bash
#!/bin/bash
kill `ps -ef | grep jenkins.war | grep -v grep | awk '{print $2}'`
```
### 服务脚本
vi /etc/init.d/jenkins
chomod a+x jenkins
```bash
#! /bin/bash
# chkconfig: 2345 90 10
# description: Jenkins Continuous Integration server
# processname: /usr/local/jenkins/jenkins.war

# Source function library.
. /etc/rc.d/init.d/functions

# Get network sysconfig.
. /etc/sysconfig/network

JENKINS_HOME=/home/service/jenkins
#生产环境请谨慎配置此选项
JENKINS_USER=root

# Check that networking is up, otherwise we can't start
[ "${NETWORKING}" = "no" ] && exit 0

# Get the Jenkins sysconfig
[ -f /etc/sysconfig/jenkins ] && . /etc/sysconfig/jenkins
[ "${JENKINS_HOME}" = "" ] &&
    JENKINS_HOME=/usr/local/jenkins
[ "${JENKINS_USER}" == "" ] &&
    JENKINS_USER=jenkins

startup=${JENKINS_HOME}/start-jenkins.sh
shutdown=${JENKINS_HOME}/stop-jenkins.sh

start(){
    echo -n $"Starting Jenkins service: "
    pid=`ps -ef | grep [j]enkins.war | wc -l`
    if [ $pid -gt 0 ]; then
        echo "Jenkins is already running"
        exit 1
    fi
    su - $JENKINS_USER -c $startup
    RETVAL=$?
    [ $RETVAL == 0 ] &&
        echo "Jenkins was started successfully." ||
        echo "There was an error starting Jenkins."
}

stop(){
    action $"Stopping Jenkins service: "
    pid=`ps -ef | grep [j]enkins.war | wc -l`
    if [ ! $pid -gt 0 ]; then
        echo "Jenkins is not running"
        exit 1
    fi
    su - $JENKINS_USER -c $shutdown
    RETVAL=$?
    [ $RETVAL == 0 ] &&
        echo "Jenkins was stopped successfully." ||
        echo "There was an error stopping Jenkins."
}

status(){
    pid=`ps -ef | grep [j]enkins.war | wc -l`
    if [ $pid -gt 0 ]; then
        echo "Jenkins is running..."
    else
        echo "Jenkins is stopped..."
    fi
}

restart(){
    stop
    sleep 5
    start
}

# Call functions as determined by args.
case "$1" in
start)
    start;;
stop)
    stop;;
status)
    status;;
restart)
    restart;;
*)
    echo $"Usage: $0 {start|stop|status|restart}"
    exit 1
esac

exit 0
```
### 大功告成
然后我们可以通过以下几个命令起停、查看jenkins状态，以及开启服务
```bash
service jenkins status
service jenkins start
service jenkins restart
service jenkins stop
chkconfig jenkins on
```
也可以在 /etc/sysconfig/jenkins 配置启动参数
```bash
# Jenkins system configuration
JENKINS_HOME=/usr/local/jenkins
JENKINS_USER=jenkins
JENKINS_LOG=/home/jenkins/jenkins.log
JENKINS_JAVA=/usr/bin/java
JENKINS_JAVAOPTS=""
JENKINS_IP=0.0.0.0
JENKINS_PORT=8080
JENKINS_ARGS=""
```

## 初始化Jenkins
假设我们配置的端口为8080
接下来我们访问Jenkins服务

1. 解锁Jenkins  
按照提示的路径将密码粘贴到输入框
![image](https://github.com/silencerv/Ops/blob/master/Jenkins/pics/unlock_jenkins.jpg)
2. 插件安装  
如图，根据需要选择安装类型，这里我们选择安装推荐的插件  
插件随时可以安装，以后有需要在自行安装
![image](https://github.com/silencerv/Ops/blob/master/Jenkins/pics/jenkins_plugin_choose.jpg)
接下来等待jenkins安装插件
![image](https://github.com/silencerv/Ops/blob/master/Jenkins/pics/step_up.png)
3. 设置管理员帐户

4. 配置JenkinsURL  
PS:如果登入jenkins为空白页,尝试重启jenkins服务即可

## 安装插件
### 安装Maven Plugin
1. 首先需要安装Maven,<a href="https://maven.apache.org/download.cgi">Download Maven</a>  
2. 进入系统管理-全局工具配置-Maven-新增Maven-填写配置-Save  
3. 系统管理-管理插件-选择可选插件tab-搜索Maven Integration-直接安装

## 创建任务
1. 点击左上角的新建任务
2. git等配置忽略，自行解决
3. 期望在执行构建前检查代码是否已经合并master，在Pre Steps执行以下脚本
```bash
#!/bin/bash
if [ ${branch} == "master" ]
then
  echo "this is master not need merge"
else
  result=`git branch -r --contains origin/master | grep "/${branch}\$" | wc -l`
  echo $result
  if [ $result == 0 ]
  then
  	echo "not merged master"
  	echo "代码尚未合并master,请手动进行合并！！！！！！！！！！！！！！！！！！！！！！！"
  exit 1
  else
  	echo "merged master"
  fi
fi
```
4. 配置Maven执行参数以及pom.xml文件位置
5. 在PostSteps执行部署脚本，变量请按需修改
```bash
#!/bin/bash
BUILD_ID=DONTKILLME
DEPLOY_HOME=/home/app/spring-boot-demo
JAR_NAME=SpringBoot
MAVEN_TARGET=target
JAVA=/home/jdk/jdk1.8.0_201/bin/java
mkdir -p ${DEPLOY_HOME}/logs

JVM_OPT="-Xmx256m -Xms256m -XX:MaxMetaspaceSize=256m -XX:+UseG1GC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:${DEPLOY_HOME}/logs/gc.log -Dserver.port=7788 -Xrunjdwp:server=y,transport=dt_socket,address=9088,suspend=n"


function init() {
	PID_NUM=$(ps aux |grep "${JAR_NAME}.jar"|grep -v "grep"|awk '{print $2}')
	echo ${PID_NUM}
	if [ ${PID_NUM} ] ; then
		echo kill ${JAR_NAME}.jar
		kill ${PID_NUM}
		echo sleep 10
		sleep 10
		PID_NUM=$(ps aux |grep "${JAR_NAME}.jar"|grep -v "grep"|awk '{print $2}')
		if [ ${PID_NUM} ] ; then
			echo kill -9 ${JAR_NAME}.jar
			kill -9 ${PID_NUM}
		fi
	else
		echo ${JAR_NAME}.jar not running
	fi
}



function run() {
	rm -f ${DEPLOY_HOME}/${JAR_NAME}.jar
	cp ${WORKSPACE}/${MAVEN_TARGET}/${JAR_NAME}-1.0-SNAPSHOT.jar ${DEPLOY_HOME}/${JAR_NAME}.jar
    nohup $JAVA ${JVM_OPT} -jar ${DEPLOY_HOME}/${JAR_NAME}.jar > ${DEPLOY_HOME}/logs/spring.out 2>&1 &
	echo run finished
}
init
run
```
6. 保存，执行脚本即可

## 远程部署
