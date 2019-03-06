podTemplate(
    name: 'questcode',
    namespace: 'devops',
    label: 'questcode', 
    containers: [
        containerTemplate(args: 'cat', name: 'docker', command: '/bin/sh -c', image: 'docker', ttyEnabled: true),
        containerTemplate(args: 'cat', name: 'helm', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:v2.11.0', ttyEnabled: true)
    ],
    volumes: [
      hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
    ]
){
    // Start Pipeline
    node('questcode') {
        def REPOS
        def IMAGE_NAME = "frontend"
        def IMAGE_VERSION
        def ENVIRONMENT = "staging"
        def GIT_REPO_URL = "git@github.com:warior-bootcamp/frontend.git"
        def CHARTMUSEUM_REPO_URL = "http://helm-chartmuseum:8080"
        stage('Checkout') {
            echo 'Iniciando Clone do Repositório'
            REPOS = checkout([$class: 'GitSCM', branches: [[name: '*/master'], [name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: GIT_REPO_URL ]]])
            IMAGE_VERSION = sh returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION = IMAGE_VERSION.trim()
        }
        stage('Package') {
            container('docker') {
                echo 'Iniciando empacotamento com Docker'
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} . --build-arg NPM_ENV='${ENVIRONMENT}'"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
                    // some block
                }
            }
        }
        stage('Deploy') {
            container('helm'){
                echo 'Iniciando Deploy com Helm'
                sh "helm init --client-only"
                sh "helm repo add questcode ${CHARTMUSEUM_REPO_URL}"
                sh "helm repo update"
                sh "helm upgrade staging-frontend questcode/${IMAGE_NAME} --set image.tag=${IMAGE_VERSION}"
            }
        }
    } // end of node
}