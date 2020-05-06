---
title: Jenkins Blue Ocean
date: 2019-05-04

categories:
    - Developer
---

基于 Docker 搭建,
附带流水线、部署脚本

<!--more-->

## Docker Run

```bash
sudo mkdir -p /data/docker/jenkins
sudo chown -R 1000:1000 /data/docker/jenkins

sudo docker run -d -p 8090:8080 -v /data/docker/jenkins:/var/jenkins_home --restart=always --name jenkins jenkinsci/blueocean
```

## Jenkins Pipeline

```bash
node {
   // maven 模块路径
   def mavenModule="gateway"

   // 服务名称
   def serverName="gateway"

   // maven build 后制品所在目录
   def sourceDirectory="${mavenModule}/target/"

   // 部署环境
   def env="prod"

   // JVM 参数
   def jvmOptions=""

   // 要部署到到主机列表 (系统配置 -> Publish Over SS -> SSH Server -> Name)
   def hosts=["187", "152"]

   // ---------------------------------------

   // 远程服务器目录
   def remoteDirectory="data/sango/server/"
   // 远程服务器部署脚本路径
   def deployShellFile="/data/sango/shell/deploy.sh"
   // 制品文件
   def sourceFiles="${sourceDirectory}${serverName}.jar"
   // 到远程服务器执行的命令(三个参数: serverName, env, jvmOptions)
   def execCommand="cd ${remoteDirectory}${serverName} && sudo sh ${deployShellFile} ${serverName} ${env} ${jvmOptions}"

   def mvnHome
   def workspace = pwd()

   stage('git pull') {
      println(workspace)

      // for display purposes
      // Get some code from a GitHub repository
      git branch: 'master', credentialsId: 'xxxx-xxxx', url: 'git repository'
   }
   stage('maven build') {
      println(workspace)

      // Get the Maven tool.
      // ** NOTE: This 'M3' Maven tool must be configured
      // ** in the global configuration.
      mvnHome = tool 'M3'

      sh "${mvnHome}/bin/mvn --version"
      // Run the maven build
      sh "${mvnHome}/bin/mvn -B clean package -pl ${mavenModule} -am -Dmaven.test.skip=true -Dautoconfig.skip"
      archiveArtifacts "${sourceFiles}"
   }
   stage('deploy') {
      println(workspace)

      script {
         // 先发布一台验证
         success = false
         while (!success) {
            try {
               sshPublisher(
                  continueOnError: false, failOnError: true,
                  publishers: [
                     sshPublisherDesc(
                        configName: hosts.pop(),
                        verbose: true,
                        transfers: [
                           sshTransfer(
                              sourceFiles: "${sourceFiles}",
                              removePrefix: "${sourceDirectory}",
                              remoteDirectory: "${remoteDirectory}/${serverName}",
                              execCommand: "${execCommand}"
                           )
                        ]
                     )
                  ]
               )
               success = true
            } catch (e) {
               println(e)
               success = false
               result = input message: '', ok: 'Retry', submitterParameter: 'operator'
            }
         }

         // 人工卡点
         result = input message: '', ok: 'Confirm', parameters: [choice(choices: ['Resume'], description: '', name: 'action')], submitterParameter: 'operator'
         if (result.action == 'Resume') {
            for (host in hosts) {
               success = false
               while (!success) {
                  try {
                     sshPublisher(
                        continueOnError: false, failOnError: true,
                        publishers: [
                           sshPublisherDesc(
                              configName: host,
                              verbose: true,
                              transfers: [
                                 sshTransfer(
                                    sourceFiles: "${sourceFiles}",
                                    removePrefix: "${sourceDirectory}",
                                    remoteDirectory: "${remoteDirectory}/${serverName}",
                                    execCommand: "${execCommand}"
                                 )
                              ]
                           )
                        ]
                     )
                     success = true
                  } catch (e) {
                     println(e)
                     success = false
                     result = input message: '', ok: 'Retry', submitterParameter: 'operator'
                  }
               }
            }
         } else {
            // 回滚至上个版本
         }
      }
   }
}
```

## Deploy Shell Script

```bash
# ServerName
SERVER=$1

# Spring Profiles Active
ENV=$2

# JVM Options
JVM_OPTIONS=$3

# Rollback
ROLLBACK=$4

# Default Value
DEFAULT_ENV="prod"
DEFAULT_JVM_OPTIONS="-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m -Xms1024m -Xmx1024m -Xmn256m -Xss256k -XX:SurvivorRatio=8 -XX:+UseConcMarkSweepGC"
STABLE='stable'

if [ ! -n "$ENV" ]; then
  ENV=$DEFAULT_ENV
fi

if [ ! -n "$JVM_OPTIONS" ]; then
  JVM_OPTIONS=$DEFAULT_JVM_OPTIONS
fi

if [ ! -n "$ROLLBACK" ]; then
  SERVER=$STABLE
fi

# kill old
PID=`ps -ef | grep ${SERVER}.jar | grep -v grep | awk '{print $2}'`
if [ 0 -ne ${#PID} ]; then
  echo "kill ${SERVER}"
  kill $PID
fi

# run new
nohup java -jar ${SERVER}.jar --spring.profiles.active=${ENV} > ${SERVER}.log 2>&1 &
echo "start server ${SERVER}:${ENV} [${JVM_OPTIONS}]"

# mark as latest
if [ ! -n "$ROLLBACK" ]; then
  cp ${SERVER}.jar ${STABLE}.jar
fi
```

## 完成效果

![jrnkind-blue.png](http://cdn.talei.me/blog/2019/jenkins-blue-ocean/jrnkind-blue.png)

![jenkins-input.png](http://cdn.talei.me/blog/2019/jenkins-blue-ocean/jenkins-input.png)

![jenkins-publish-over-ssh.png](http://cdn.talei.me/blog/2019/jenkins-blue-ocean/jenkins-publish-over-ssh.png)

## Git 仓库过大的问题

默认十分钟超时, 手动 clone 即可.

```bash
sudo docker exec -it jenkins /bin/bash
cd /var/jenkins_home/workspace/
```

## 参考资料

- [官方资料](https://www.jenkins.io/zh/doc/book/blueocean/getting-started/)
- [docker hub/blueocean](https://hub.docker.com/r/jenkinsci/blueocean)
- [Intro to Jenkins Pipelines and Publishing Over SSH](https://dzone.com/articles/intro-to-jenkins-pipeline-and-using-publish-over-s)
