// template de pod que será utilizado no pepiline
def LABEL_ID = "qestcode-${UUID.randomUUID().toString()}"
podTemplate(
    name: 'questcode',
    namespace: 'devops',
    label: LABEL_ID, 
    containers: [
        // imagem docker
        containerTemplate(args: 'cat', name: 'docker', command: '/bin/sh -c', image: 'docker', ttyEnabled: true),
        // imagem helm
        containerTemplate(args: 'cat', name: 'helm', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:v2.11.0', ttyEnabled: true)
    ],
    volumes: [
        // volume que precisa ser criado para utilização do docker que
        // procura pelo docker.sock sempre 
      hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
    ]
){
    // variáveis que serão utilizadas no decorrer da pipeline
    def REPOS
    def IMAGE_NAME = "frontend"
    def IMAGE_VERSION
    def IMAGE_POXFIX = ""
    def KUBE_NAMESPACE
    def ENVIRONMENT
    def GIT_REPO_URL = "git@github.com:warior-bootcamp/frontend.git"
    def GIT_BRANCH
    def HELM_CHART_NAME = "questcode/frontend"
    def HELM_DEPLOY_NAME
    def CHARTMUSEUM_REPO_URL = "http://helm-chartmuseum:8080"
    def NODE_PORT = "30080"
    
    // Start Pipeline
    node(LABEL_ID) {    
        
        // stages = cada estágio da pipeline
        // stage de checagem de alguns parâmetros
        stage('Checkout') {
            echo 'Iniciando Clone do Repositório'

            // acesso ao repositório do github
            REPOS = checkout([$class: 'GitSCM', branches: [[name: '*/master'], [name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: GIT_REPO_URL ]]])
            
            // pegando a branch que vou utilizar
            GIT_BRANCH = REPOS.GIT_BRANCH
            
            // variáveis serão setadas de acordo com a branch: master ou develop
            // se for uma branch de feature sairá do pipeline
            if(GIT_BRANCH.equals("origin/master")){
                KUBE_NAMESPACE = "prod"
                ENVIRONMENT = "production"
            }else if(GIT_BRANCH.equals("origin/develop")){
                KUBE_NAMESPACE = "staging"
                ENVIRONMENT = "staging"
                IMAGE_POXFIX = "-RC"
                NODE_PORT = "31080"
            }else{
                def error = "Não existe pipeline para a branch ${GIT_BRANCH}"
                echo error
                throw new Exception(error)
            }
            
            // setando variáveis de Helm e numero de versão da aplicação
            HELM_DEPLOY_NAME = KUBE_NAMESPACE + "-frontend"
            IMAGE_VERSION = sh returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION = IMAGE_VERSION.trim() + IMAGE_POXFIX
        }
        
        // stage de empacotamento da aplicação gerando a imagem docker e subindo no dockerhub
        stage('Package') {
            container('docker') {
                echo 'Iniciando empacotamento com Docker'
                
                // buscando credenciais do docker gravadas no jenkins,
                // logando no docker para poder subir imagem para dockerhub
                // gerando build da imagem e subindo no dockerhub
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} . --build-arg NPM_ENV='${ENVIRONMENT}'"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
                }
            }
        }

        // stage de deploy da aplicação com Helm
        stage('Deploy') {
            container('helm'){
                echo 'Iniciando Deploy com Helm'

                // inicializando helm
                sh "helm init --client-only"

                // adicionando nosso repositório chart museum
                sh "helm repo add questcode ${CHARTMUSEUM_REPO_URL}"

                // update nos repositórios
                sh "helm repo update"

                //fazendo upgrade ou primeira instalação da aplicação
                try{
                    // fazer helm upgrade
                    sh "helm upgrade --namespace=${KUBE_NAMESPACE} ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION} --set service.nodePort=${NODE_PORT}"
                }catch(Exception  e){
                    //fazer helm install
                    sh "helm install --namespace=${KUBE_NAMESPACE} --name ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION} --set service.nodePort=${NODE_PORT}"
                }
            }
        }
    } // end of node
}