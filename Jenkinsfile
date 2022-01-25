#!/usr/bin/env groovy

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }

    tools {
        jdk 'openjdk-13'
        maven 'maven 3.8.4'
        dockerTool 'docker-latest'
    }

    environment {
        POM_VERSION = getVersion()
        JAR_NAME = getJarName()
        AWS_ECR_REGION = 'us-east-1'
        AWS_ECR_URL = '760761285600.dkr.ecr.us-east-1.amazonaws.com/devops-test-jenkins-ecs'
        AWS_ECS_CLUSTER = 'DevOps-Test-ECS'
        AWS_ECS_SERVICE = 'DevOps-Test-ECS-Test-Service'
        AWS_ECS_TASK_DEFINITION = 'DevOps-Test-ECS-Test-TaskDef'
        AWS_ECS_COMPATIBILITY = 'FARGATE'
        AWS_ECS_NETWORK_MODE = 'awsvpc'
        AWS_ECS_CPU = '256'
        AWS_ECS_MEMORY = '512'
        AWS_ECS_TASK_DEFINITION_PATH = './ecs/container-definition-update-image.json'
        AWS_ECS_EXECUTION_ROLE = 'ecsTaskExecutionRole'
    }

    stages {
        stage('Build & Test') {
            steps {
                echo 'Build & Test'
                withMaven(options: [artifactsPublisher(), mavenLinkerPublisher(), dependenciesFingerprintPublisher(disabled: true), jacocoPublisher(disabled: true), junitPublisher(disabled: true)]) {
                    sh "mvn -B -U clean package"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Build Docker Image'
                script {
                    docker.build("${AWS_ECR_URL}:${POM_VERSION}", "--build-arg JAR_FILE=${JAR_NAME} .")
                }
            }
        }

        stage('Push image to ECR') {
            steps {
                echo 'Push image to ECR'
                withAWS(region: "${AWS_ECR_REGION}", credentials: 'AWS_USER') {
                    script {
                        def login = ecrLogin()
                        sh('#!/bin/sh -e\n' + "${login}") // hide logging
                        docker.image("${AWS_ECR_URL}:${POM_VERSION}").push()
                    }
                }
            }
        }

        stage('Deploy in ECS') {
            steps {
                echo 'Deploy in ECS'
                script {
                    updateContainerDefinitionJsonWithImageVersion()
                    sh("/usr/local/bin/aws ecs register-task-definition --region ${AWS_ECR_REGION} --family ${AWS_ECS_TASK_DEFINITION} --execution-role-arn ${AWS_ECS_EXECUTION_ROLE} --requires-compatibilities ${AWS_ECS_COMPATIBILITY} --network-mode ${AWS_ECS_NETWORK_MODE} --cpu ${AWS_ECS_CPU} --memory ${AWS_ECS_MEMORY} --container-definitions file://${AWS_ECS_TASK_DEFINITION_PATH}")
                    // def taskRevision = sh(script: "/usr/local/bin/aws ecs describe-task-definition --task-definition ${AWS_ECS_TASK_DEFINITION} | egrep \"revision\" | tr \"/\" \" \" | awk '{print \$2}' | sed 's/\"\$//'", returnStdout: true)
                    sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}")
                }
            }
        }
    }

    post {
        always {
            echo 'Post'
            junit allowEmptyResults: true, testResults: 'target/surfire-reports/*.xml'
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'target/site/jacoco-ut/', reportFiles: 'index.html', reportName: 'Unit Testing Coverage', reportTitles: 'Unit Testing Coverage'])
            jacoco(execPattern: 'target/jacoco-ut.exec')
            deleteDir()
            sh "docker rmi ${AWS_ECR_URL}:${POM_VERSION}"
        }
    }
}

def getJarName() {
    def jarName = getName() + '-' + getVersion() + '.jar'
    echo "jarName: ${jarName}"
    return  jarName
}

def getVersion() {
    def pom = readMavenPom file: './pom.xml'
    return pom.version
}

def getName() {
    def pom = readMavenPom file: './pom.xml'
    return pom.name
}

def updateContainerDefinitionJsonWithImageVersion() {
    def containerDefinitionJson = readJSON file: AWS_ECS_TASK_DEFINITION_PATH, returnPojo: true
    containerDefinitionJson[0]['image'] = "${AWS_ECR_URL}:${POM_VERSION}".inspect()
    echo "task definiton json: ${containerDefinitionJson}"
    writeJSON file: AWS_ECS_TASK_DEFINITION_PATH, json: containerDefinitionJson
}
