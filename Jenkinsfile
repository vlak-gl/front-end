pipeline {

  parameters {
    choice(
      choices: ['front-end' , 'catalogue' , 'shipping'],
      description: 'What servicename?',
      name: 'servicename')
    choice(
      choices: ['ServiceUpgrade' , 'ServiceAdd'],
      description: '',
      name: 'REQUESTED_ACTION')
    choice(
      choices: ['vlk', 'default'],
      description: '',
      name: 'namespace')    
    choice(
      choices: ['dev', 'mgt', 'test1'],
      description: '',
      name: 'cluster')
    string(defaultValue: "vlak-gl", description: '', name: 'git_name')
    string(defaultValue: "origin/master", description: '', name: 'git_branch')
    }    
    
  agent {
    kubernetes {
      label 'mypod'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: some-label-value
spec:
  containers:
  - name: helm
    image: dtzar/helm-kubectl:2.11.0
    command:
    - cat
    tty: true
  - name: docker
    image: docker:1.11
    command:
    - cat
    tty: true

    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
    }
  }
  stages {
    stage ('Prepare') {
      steps {
        sh 'echo "preparing...."'
        checkout([$class: 'GitSCM',
          branches: [[name: "${git_branch}"]],
          doGenerateSubmoduleConfigurations: false,
          extensions: [[$class: 'LocalBranch']],
          submoduleCfg: [],
          userRemoteConfigs: [[
            credentialsId: 'git-main-frontend',
            url: 'git@github.com:${git_name}/${servicename}.git']]])
       }                  
    }
    stage('Docker build and publish') {
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: 'docker_registry_globallogicpractices',
            usernameVariable: 'DOCKER_REGISTRY_USERNAME',passwordVariable: 'DOCKER_REGISTRY_PASSWORD')]) {
              sh '''
                export IMAGE_WITH_TAG="globallogicpractices/opengine-base:${servicename}-app-${BUILD_NUMBER}"                                        
                docker login --username "$DOCKER_REGISTRY_USERNAME" --password "$DOCKER_REGISTRY_PASSWORD"
                docker build -t opengine-${servicename} .
                docker tag opengine-${servicename} ${IMAGE_WITH_TAG}
                docker push ${IMAGE_WITH_TAG}
                docker rmi ${IMAGE_WITH_TAG}               
              '''
          }
        }
      }
    }
    stage('Deploy') {
      steps {  
        container('helm') {
          withCredentials([file(credentialsId: "kube-config-vlk-${cluster}-cluster", variable: 'KUBECONFIG'),
            usernamePassword(credentialsId: 'docker_registry_globallogicpractices',
            usernameVariable: 'DOCKER_REGISTRY_USERNAME',passwordVariable: 'DOCKER_REGISTRY_PASSWORD')]) {
              sh ''' 
                export IMAGE="globallogicpractices/opengine-base"
                if [ "${REQUESTED_ACTION}" == "AddService" ];then
                  helm install --name ${servicename} helm-chart --set name=${servicename},image.tag=${servicename}-app-${BUILD_NUMBER},image.repository=${IMAGE},service.type=ClusterIP,service.port=8020 --namespace ${namespace}
                else
                  helm upgrade ${servicename} helm-chart --set name=${servicename},image.tag=${servicename}-app-${BUILD_NUMBER},image.repository=${IMAGE},service.type=ClusterIP,service.port=8020 --namespace ${namespace}    --recreate-pods
                fi
              '''
          }
        }
      }
    }


  }
}