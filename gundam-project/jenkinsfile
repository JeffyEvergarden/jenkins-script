#!groovy

// 用到的插件：NodeJS、pipeline、checkbox、git、SCM、SSH、pipeline-steps

// 给 remote 定义变量
def GetRemoteServer(name, host, credentialsId){
    def remote = [:]
    remote.name = name
    remote.host = host
    remote.port = 22
    remote.timeoutSec = 300000
    remote.allowAnyHosts = true
    //通过withCredentials调用Jenkins凭据中已保存的凭据，credentialsId需要填写，其他保持默认即可
    withCredentials([usernamePassword(credentialsId: credentialsId, passwordVariable: 'password', usernameVariable: 'userName')]) {
        remote.user = "${userName}"
        remote.password = "${password}"
    }
    return remote
}



def generateStage(projectEnv) {
    // return {
        stage("build") {
            // 远程服务器配置
            // 得到服务器定义对象
            // 根据凭据获取112服务器
            def remote = GetRemoteServer('11.113.1.50', '11.113.1.50', 'jeffy-liang-50')

            def uatRemote = GetRemoteServer('11.112.0.98', '11.112.0.98', 'jeffy-liang-98')
            
            // 打包
            if (projectEnv == 'default') {
                println "执行构建  npm run build"
                sh "npm run build"
            } else {
                println "构建 ${projectEnv}"
                sh "npm run build:${projectEnv}"
            }            

            // 进入dist目录打包
            dir("dist") {
            sh "tar -cvzf gundam-${projectEnv}.tar.gz *"
            }
            // 移到./路径
            sh """
                rm -rf gundam-${projectEnv}.tar.gz
                mv  dist/gundam-${projectEnv}.tar.gz  ./
                """
                
            println "发布"
            // linux 脚本
            if (packageOnEnv == 'SIT') {
              println "操作地址: ${remote.host} /data/robot/robot_chatWeb/static/"
              sshCommand remote: remote, command: """
                  if [ ! -d "/home/appuser/cooperation-${projectEnv}" ];then
                      mkdir -p /home/appuser/cooperation-${projectEnv}
                  fi
                  """
              sshPut remote: remote, from: "./gundam-${projectEnv}.tar.gz", into: "/home/appuser/cooperation-${projectEnv}/"
              sshCommand remote: remote, command: """
                  cd /home/appuser/cooperation-${projectEnv}
                  tar -xvf gundam-${projectEnv}.tar.gz
                  rm -rf gundam-${projectEnv}.tar.gz
                  rm -rf  /data/robot/robot_chatWeb/static/*
                  mv ./*  /data/robot/robot_chatWeb/static/
                  """
            } else if (packageOnEnv == 'UAT') {
              println "操作地址: ${uatRemote.host} /data/robot/airobot/youcash-aip-robot-admin/static"
              sshCommand remote: uatRemote, command: """
                  if [ ! -d "/home/appuser/cooperation-${projectEnv}" ];then
                      mkdir -p /home/appuser/cooperation-${projectEnv}
                  fi
                  """
              sshPut remote: uatRemote, from: "./gundam-${projectEnv}.tar.gz", into: "/home/appuser/cooperation-${projectEnv}/"
              sshCommand remote: uatRemote, command: """
                  cd /home/appuser/cooperation-${projectEnv}
                  tar -xvf gundam-${projectEnv}.tar.gz
                  rm -rf gundam-${projectEnv}.tar.gz
                  rm -rf  /data/robot/airobot/youcash-aip-robot-admin/static/*
                  mv ./*  /data/robot/airobot/youcash-aip-robot-admin/static/
                  """
            }
            
        }
    // }
}

pipeline {
    agent any
    //agent {label "slave-2"}
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 25, unit:'MINUTES') // 25分钟超时
    }

    environment {
        NODE_VERSION = 'NodeJS14'
    }

    parameters{
        choice(name: 'projectEnv', choices: ['default'], description: '请选择前端打包环境 npm run build方式')
        gitParameter branchFilter: 'origin/(.*)', defaultValue: 'master', name: 'branch', type: 'PT_BRANCH'
        choice(name: 'packageOnEnv', choices: ['SIT', 'UAT'], description: '请选择部署环境')
    }

    stages {

        // 拉代码
        stage('init') {
            steps {
                script {
                    echo "当前部署环境：${projectEnv}"
                    // echo "构建构建项目：${projects}"
                    echo "开始构建"
                    git credentialsId: '2f90cfac-bd3c-4e60-8770-407fe57d4310', url: "http://172.168.0.14:8181/gitlab/datacenter/gundam-platform.git", branch: "${branch}"
                }
            }
        }
        // npm install
        stage('install') {
            tools {
                nodejs "${NODE_VERSION}"
            }
            steps {
                script {
                    sh "date"
                    echo "node && npm --version"
                    sh "node --version;npm --version"
                    // sh "rm -rf ./node_modules"
                    sh """
                        if [ ! -d "./node_modules" ];then
                          echo 'has node_modules dir'
                        else 
                          echo 'not has node_modules dir'
                        fi
                    """
                    echo "npm install"
                    sh "npm install"
                }
            }
        }

        stage('build') {
            steps {
                script {
                    generateStage(projectEnv)
                    echo "After build"
                    sh "date"
                }
            }
        }

    }
}