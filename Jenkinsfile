def label = "kubectl-${UUID.randomUUID().toString()}"

podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
spec:
  containers:

  - name: helm
    image: dtzar/helm-kubectl:2.13.0
    command: ['cat']
    tty: true
  - name: maven
    image: maven:3.6.1-jdk-8-slim
    command: ['cat']
    tty: true
  - name: docker
    image: docker:1.11
    command: ['cat']
    tty: true

    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
  ){

node(label) {

    parameters {
        choice(
            choices: ['Build', 'AddService', 'Deploy'],
            description: '',
            name: 'REQUESTED_ACTION')
        string(defaultValue: "niko-dev", description: 'What environment?', name: 'namespace')
        string(defaultValue: "productcatalogue", description: 'What servicename?', name: 'servicename')
        string(defaultValue: "latest", description: 'What buildVersion?', name: 'buildVersion')
    }

        stage ('Prepare') {
                echo "preparing...."
                checkout([$class: 'GitSCM',
                     branches: [[name: "origin/master"]],
                     doGenerateSubmoduleConfigurations: false,
                     extensions: [[$class: 'LocalBranch']],
                     submoduleCfg: [],
                     userRemoteConfigs: [[
                         credentialsId: 'git-akopachevskyy-globallogic',
                         url: 'git@github.com:vlak-gl/OpenGine-${servicename}-app.git']]])
        }

        stage('Build') {

          container('maven') {
              withCredentials([usernamePassword(credentialsId: 'docker_registry_credentials',
                              usernameVariable: 'DOCKER_REGISTRY_USERNAME',passwordVariable: 'DOCKER_REGISTRY_PASSWORD')]) {
                sh "mvn clean install"
              }
          }
        }


        def IMAGE_WITH_TAG = "globallogicpractices/opengine-base:${servicename}-app-${BUILD_NUMBER}"
        stage('Docker build and publish') {

            container('docker') {
                withCredentials([usernamePassword(credentialsId: 'docker_registry_credentials',
                                usernameVariable: 'DOCKER_REGISTRY_USERNAME',passwordVariable: 'DOCKER_REGISTRY_PASSWORD')]) {
                    sh '''
                        echo REQUESTED_ACTION: ${REQUESTED_ACTION}
                        export IMAGE_WITH_TAG="globallogicpractices/opengine-base:${servicename}-app-${BUILD_NUMBER}"
                        if [ "AddService" == "${REQUESTED_ACTION}" ] || [ "Build" == "${REQUESTED_ACTION}" ] ; then
                          docker login --username "$DOCKER_REGISTRY_USERNAME" --password "$DOCKER_REGISTRY_PASSWORD"
                          docker build -t opengine-${servicename} .
                          docker tag opengine-${servicename} ${IMAGE_WITH_TAG}
                          docker push ${IMAGE_WITH_TAG}
                          docker rmi ${IMAGE_WITH_TAG}
                        fi
                      '''
                }
            }
        }

        stage('Deploy') {

            container('helm') {

               withCredentials([file(credentialsId: 'kube-config', variable: 'KUBECONFIG'),
                                usernamePassword(credentialsId: 'docker_registry_credentials',
                                usernameVariable: 'DOCKER_REGISTRY_USERNAME',passwordVariable: 'DOCKER_REGISTRY_PASSWORD')]) {
                     sh ''' export IMAGE="globallogicpractices/opengine-base"
                     if [ "${REQUESTED_ACTION}" == "AddService" ];then
                     helm install --name ${servicename}  Charts   --set name=${servicename},image.tag=${servicename}-app-${BUILD_NUMBER},image.repository=${IMAGE},service.type=ClusterIP,service.port=8020 --namespace ${namespace}
                     else
                     helm upgrade ${servicename} Charts   --set name=${servicename},image.tag=${servicename}-app-${BUILD_NUMBER},image.repository=${IMAGE},service.type=ClusterIP,service.port=8020 --namespace ${namespace}    --recreate-pods
                     #helm upgrade ${servicename} Charts   --set name=${servicename},image.tag=latest,image.repository=${IMAGE_WITH_TAG},service.type=${servicetype},service.port=${port} --namespace ${namespace}    --recreate-pods
                     fi
                     '''
                       }
             }
         }
    }
}
