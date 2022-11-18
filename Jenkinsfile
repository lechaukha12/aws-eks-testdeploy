pipeline {
    environment {
        REGISTRY = "registry.f5seconds.vn"
        REGISTRY_CRED = "tiennn-registry-id"
        ENV = "$params.DEPLOYMENT_ENV"
        STAGE = "$params.DEPLOYMENT_TYPE"
        IMAGE_NAME = "$REGISTRY/$ENV/$K8S_APP_NAME"
        IMAGE_NAME_WITH_TAG = "$REGISTRY/$ENV/$K8S_APP_NAME:$BUILD_NUMBER"
        K8S_DEPLOYER_CRED = "$params.HOSTS_TARGET"
        K8S_APP_NAME = "test"
        K8S_APP_CONTEXT = "test1"
        K8S_CONTAINER_PORT = "8080"
        K8S_REG_SECRET = "f5s-registry"
        K8S_NAMESPACE = "ptud"
    }
    
    agent any
    
    stages {
        stage('Build, Tag, and Push Image') {
            when {
                anyOf {
                    expression { params.DEPLOYMENT_TYPE == 'BUILD_ONLY' }
                    expression { params.DEPLOYMENT_TYPE == 'BUILD_DEPLOY' }
                    expression { params.DEPLOYMENT_TYPE == 'UPDATE_IMAGE' }
                }
            }
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'registry_deployer_id', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')
                ]) {
                    sh 'docker login $REGISTRY --username $USERNAME --password $PASSWORD'
                }

                sh 'docker build --no-cache -t $IMAGE_NAME_WITH_TAG --build-arg REACT_APP_PREFIX_URL=$K8S_APP_CONTEXT -f  k8s/test/test/Dockerfile.' + ENV + ' .'
                sh 'docker tag $IMAGE_NAME_WITH_TAG $IMAGE_NAME:latest'
                sh 'docker push $IMAGE_NAME_WITH_TAG'
                sh 'docker push $IMAGE_NAME:latest'
                sh 'docker rmi $IMAGE_NAME_WITH_TAG'
                sh 'docker rmi $IMAGE_NAME:latest'
            }
        }

        stage('Generate deployment manifests') {
            when {
                anyOf {
                    expression { params.DEPLOYMENT_TYPE == 'BUILD_DEPLOY' }
                    expression { params.DEPLOYMENT_TYPE == 'DEPLOY_ONLY' }
                    expression { params.DEPLOYMENT_TYPE == 'MANIFEST_ONLY' }
                    expression { params.DEPLOYMENT_TYPE == 'DELETE' }
                }
            }
            steps {
                ansiColor('xterm'){
                    ansiblePlaybook colorized: true,
                    credentialsId: "$K8S_DEPLOYER_CRED", 
                    playbook: 'k8s/test/test/playbook-manifests.yaml',
                    extraVars: [
                        target_hosts: "$K8S_DEPLOYER_CRED"
                    ]
                }
            }
        }

        stage('Deploy to k8s Cluster') {
            when {
                anyOf{
                    expression { params.DEPLOYMENT_TYPE == 'BUILD_DEPLOY' }
                    expression { params.DEPLOYMENT_TYPE == 'DEPLOY_ONLY' }
                }
            }
            steps {
                ansiColor('xterm'){
                    ansiblePlaybook colorized: true,
                    credentialsId: "$K8S_DEPLOYER_CRED", 
                    playbook: 'k8s/test/test/playbook-deployment.yaml',
                    extraVars: [
                        target_hosts: "$K8S_DEPLOYER_CRED"
                    ]
                }
            }
        }

        stage('Roll out update image to k8s Cluster') {
            when {
                expression { params.DEPLOYMENT_TYPE == 'UPDATE_IMAGE' }
            }
            steps {
                sh 'kubectl --kubeconfig=k8s/test/test/kubeconfig-' + ENV + ' -n ' + K8S_NAMESPACE + ' rollout restart deployment ' + K8S_APP_NAME
            }
        }

        stage('Update deployment') {
            when {
                expression { params.DEPLOYMENT_TYPE == 'UPDATE_CONFIG' }
            }
            steps {
                ansiColor('xterm'){
                    ansiblePlaybook colorized: true,
                    credentialsId: "$K8S_DEPLOYER_CRED", 
                    playbook: 'k8s/test/test/playbook-update.yaml',
                    extraVars: [
                        target_hosts: "$K8S_DEPLOYER_CRED"
                    ]
                }
            }
        }

        stage('Delete deployment') {
            when {
                expression { params.DEPLOYMENT_TYPE == 'DELETE' }
            }
            steps {
                ansiColor('xterm'){
                    ansiblePlaybook colorized: true,
                    credentialsId: "$K8S_DEPLOYER_CRED", 
                    playbook: 'k8s/test/test/playbook-delete.yaml',
                    extraVars: [
                        target_hosts: "$K8S_DEPLOYER_CRED"
                    ]
                }
            }
        }
    }

    post {
        success {
            withCredentials([
                string(credentialsId: 'telegramBotToken', variable: 'BOT_TOKEN'),
                string(credentialsId: 'telegramChatId', variable: 'CHAT_ID')
            ]) {
                sh 'curl -s -X POST https://api.telegram.org/bot${BOT_TOKEN}/sendMessage \
                    -d chat_id=${CHAT_ID} -d parse_mode=HTML \
                    -d text="<b>*** Jenkins Pipeline ***</b> \n <b>Project</b>: ${JOB_NAME} \n <b>Stage</b>: ${STAGE} \n <b>Enviroment</b>: ${ENV} \n <b>Target</b>: ${K8S_DEPLOYER_CRED} \n <b>Status</b>: OK"'
            }
        }
        cleanup {
            cleanWs(cleanWhenNotBuilt: false,
                    deleteDirs: true,
                    disableDeferredWipeout: true,
                    notFailBuild: true,
                    patterns: [[pattern: '.gitignore', type: 'INCLUDE'],
                               [pattern: '.propsfile', type: 'EXCLUDE']])
        }
    }
}