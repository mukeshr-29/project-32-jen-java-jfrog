pipeline{
    agent any
    tools{
        jdk 'jdk17'
        maven 'mvn'
    }
    environment{
        SCANNER_HOME=tool 'sonar-scanner'
        APP_NAME="java-project"
        RELEASE="1.0.0"
        DOCKER_USER="mukeshr29"
        DOCKER_PASS='dockerhub'
        IMAGE_NAME="${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG="${RELEASE}-${BUILD_NUMBER}"
    }
    stages{
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('git checkout'){
            steps{
               git branch: 'main', url: 'https://github.com/mukeshr-29/project-32-jen-java-jfrog.git' 
            }
        }
        stage('unit testing'){
            steps{
                dir('webapp'){
                    sh 'mvn test'
                }
            }
        }
        stage('build package'){
            steps{
                dir('webapp'){
                    sh 'mvn package'
                }
            }
        }
        stage('static code analysis'){
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token', installationName: 'sonar-server'){
                        dir('webapp'){
                            sh 'mvn -U clean install sonar:sonar'
                        }
                    }
                }
            }
        }
        stage('quality gate'){
            steps{
                script{
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('jfrog artifactory configuration'){
            steps{
                rtServer(
                    id: 'jfrog-server',
                    url: 'http://18.209.12.73:8082/artifactory',
                    credentialsId: 'jfrog'
                )
                rtMavenDeployer(
                    id: 'MAVEN_DEPLOYER',
                    serverId: 'jfrog-server',
                    releaseRepo: 'libs-release-local',
                    snapshotRepo: 'libs-snapshot-local'
                )
                rtMavenResolver(
                    id: 'MAVEN_RESOLVER',
                    serverId: 'jfrog-server',
                    releaseRepo: 'libs-release',
                    snapshotRepo: 'libs-snapshot'
                )
            }
        }
        stage('deploy artifacts to jfrog'){
            steps{
                rtMavenRun(
                    tool: 'mvn',
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: 'MAVEN_DEPLOYER',
                    resolverId: 'MAVEN_RESOLVER'
                )
            }
        }
        stage('publish artifacts'){
            steps{
                rtPublishBuildInfo(
                    serverId: 'jfrog-server'
                )
            }
        }
        stage('trivy filescan'){
            steps{
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('docker build and tag'){
            steps{
                script{
                    docker.withRegistry('',DOCKER_PASS){
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('trivy image scan'){
            steps{
                sh 'trivy image mukeshr29/java-project > trivyimg.txt'
            }
        }
        stage('built docker img push'){
            steps{
                script{
                    docker.withRegistry('',DOCKER_PASS){
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage('cleanup artifacts'){
            steps{
                script{
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
        stage('deploy to eks'){
            steps{
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: ''){
                        sh 'kubectl apply -f deployment.yml'
                        sh 'kubectl apply -f service.yml'
                        sh 'kubectl rollout restart deployment.apps/java-project-deployment'
                    }
                }
            }
        }
    }
    // post{
    //     always{
    //         emailtext attachLog: true,
    //         subject: "'${currentBuild.result}'",
    //         body: "project: ${env.JOB_NAME}<br/>" +
    //             "Build Number: ${env.BUILD_NUMBER}<br/>" +
    //             "URL: ${env.BUILD_URL}<br/>",
    //         to: 'mukeshr2911@gmail.com',
    //         attachmentsPattern: 'trivyfs.txt, trivyimg.txt'
    //     }
    // }
}