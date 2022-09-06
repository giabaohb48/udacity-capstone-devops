// The user jenkins needs to be added to the group docker: sudo usermod -a -G docker jenkins
// use double quotes to reference variables
// Validate Jenkinfiles in VSCode: https://llu.is/validate-jenkinfiles-in-vscode/
pipeline {
    agent any 
    // add a new agent: https://medium.com/@_oleksii_/how-to-deploy-jenkins-agent-and-connect-it-to-jenkins-master-in-microsoft-azure-ffeb085957c0
    environment {
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'udacity-devops-capstone'
        DOCKER_IMAGE_NAME = 'udacity-devops-capstone'
        AWS_PROFILE = 'default'
    }
    stages {
        stage('verify the build system') {
            steps {
                sh 'pwd'
                sh 'ls -la'
                echo 'verify python environment'
                sh 'python3 --version'
                sh 'pip3 --version'
                withEnv(["HOME=${env.WORKSPACE}"]) {
                    sh 'printenv'
                }  // see https://stackoverflow.com/a/51688905
            }
        }
        stage('verify aws-cli v2, eksctl, kubectl') {
            steps {
                //sh 'ansible --version'
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}", profile:'default') {
                    sh 'aws --version'
                    sh 'eksctl version'
                    sh 'kubectl version --short --client'
                }  // see https://support.cloudbees.com/hc/en-us/articles/360027893492-How-To-Authenticate-to-AWS-with-the-Pipeline-AWS-Plugin
            }
        }
        stage('lint') {
            steps {
                dir("myapp"){
                    //sh 'pwd'
                    //sh 'cat Makefile'
                    sh """
                    python3 -m venv devops
                    . devops/bin/activate
                    pwd
                    make install
                    make lint
                    """ // https://stackoverflow.com/a/40937525
                }  // see https://stackoverflow.com/a/52372748
            }
        }
        stage('build docker image') {
            steps {
                dir("myapp"){
                    sh 'docker build --tag=${DOCKER_IMAGE_NAME} .'
                }
            }
        }
        stage('test the docker container') {
            steps {
                sh 'docker image ls'
                sh 'docker container ls'
                sh 'docker run -d -p 80:80 ${DOCKER_IMAGE_NAME}'
                sh 'sleep 1s'
                sh 'curl http://localhost/'
                sh 'docker stop $(docker ps -a -q)'
                sh 'docker rm -f $(docker ps -a -q)'
                sh 'docker container ls'
            }
        }
        stage('push docker image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) { 
                    sh 'docker tag ${DOCKER_IMAGE_NAME} giabaohb48/${DOCKER_IMAGE_NAME}'
                    sh 'docker push giabaohb48/${DOCKER_IMAGE_NAME}'
                }  // see https://devops4solutions.com/publish-docker-image-to-dockerhub-using-jenkins-pipeline/
                sh 'docker rmi -f $(docker images -q)'
                sh 'docker image ls'
                //sh 'docker system prune'
            }
        }
        stage('deploy to AWS EKS') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    script {
                        // determine whether the AWS EKS cluster ARN exists
                        def EKS_ARN = sh(
                            script: "aws cloudformation list-exports --query \"Exports[?Name=='eksctl-${EKS_CLUSTER_NAME}-cluster::ARN'].Value\" --output text",
                            returnStdout: true
                        ).trim()
                        // create the AWS EKS cluster by eksctl if its ARN does not exist
                        if (EKS_ARN.isEmpty()) {
                            sh """
                            eksctl create cluster --name ${EKS_CLUSTER_NAME} \
                                                  --nodegroup-name standard-workers \
                                                  --node-type t2.medium \
                                                  --nodes 2 \
                                                  --nodes-min 1 \
                                                  --nodes-max 2 \
                                                  --node-ami auto \
                                                  --region ${AWS_REGION}
                            """
                            sh 'sleep 2m'  // wait for creation
                            // update the value of EKS_ARN after the cluster is created
                            EKS_ARN = sh(
                                script: "aws cloudformation list-exports --query \"Exports[?Name=='eksctl-${EKS_CLUSTER_NAME}-cluster::ARN'].Value\" --output text",
                                returnStdout: true
                            ).trim()
                        }
                        sh 'aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME}'
                        sh "kubectl config use-context ${EKS_ARN}"
                    }
                    sh 'kubectl config current-context'
                    //sh 'kubectl delete -f deploy-k8s.yml'  // https://stackoverflow.com/a/41095466
                    sh 'kubectl apply -f deploy-k8s.yml'
                    sh 'kubectl rollout restart deployments/simple-web-app'
                    sh 'sleep 2m'  // wait for image pulling
                    sh 'kubectl get nodes'
                    sh 'kubectl get deployments'
                    sh 'kubectl get pod -o wide'
                    sh 'kubectl get service/simple-web-app'
                }
            }
        }
        stage('verify deployment') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    script {
                        def EKS_HOSTNAME = sh(
                            script: 'kubectl get svc simple-web-app -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"',
                            returnStdout: true
                            ).trim()
                        sh "curl ${EKS_HOSTNAME}:8080"  // see https://stackoverflow.com/a/40078286
                    }
                }
            }
        }
        stage('check rollout') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh 'kubectl rollout status deployments/simple-web-app'
                }
            }
        }
    }
}