pipeline {
  agent {
    kubernetes {
        label 'docker-build-pod'
        yamlFile 'podTemplate/jw-workshop-docker-build.yaml'
    }
  }
  stages {
    stage('Docker Build') {
      steps {
        container('docker'){
          sh 'docker build -t bbjoern/jw-workshop:latest .'
        }
      }
    }
    stage('Docker Push') {
      steps {
        container('docker'){
          withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
            sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPassword}"
            sh 'docker push bbjoern/jw-workshop:latest'
          }
        }
      }
    }
    stage('Deploy to VKE cluster') {
      steps {
        container('vke-kubectl'){
          withCredentials([usernamePassword(credentialsId: 'VCS', usernameVariable: 'orgID', passwordVariable: 'apiToken')]) {
            sh "vke account login -t ${env.orgID} -r ${env.apiToken}"
            sh '''
                 vke cluster merge-kubectl-auth bbrundert-devops
		 kubectl delete namespace jw-workshop || true
                 sleep 5
                 kubectl create namespace jw-workshop
		 kubectl create -n jw-workshop -f deployFiles/deployment.yaml
		 kubectl create -n jw-workshop -f deployFiles/service.yaml
                 echo "Node Server Launched!"
            '''
          }
        }
      }
    }
  }
}
