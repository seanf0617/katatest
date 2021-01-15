def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
def loadValuesYaml(x){
  def valuesYaml = readYaml (file: './pipeline.yml')
  return valuesYaml[x];
}

pipeline {
    environment {
        
	    //credentials
	    dockerHubCredential = loadValuesYaml('dockerHubCredential')
            awsCredential = loadValuesYaml('awsCredential')
	    
	    //docker config
	    imageName = loadValuesYaml('imageName')
	    slackChannel = loadValuesYaml('slackChannel')
	    dockerImage = ''
	    
	    //s3 config
            backendFile = loadValuesYaml('backendFile')
            backendPath = loadValuesYaml('backendPath')
	    
	    //additional external feedback
	    successAction = loadValuesYaml('successAction')
	    failureAction = loadValuesYaml('failureAction')  
	    app_url = ''
	    
   }
    agent any
    stages {
        stage('Build Node App') {
            steps {
                echo 'Building Node app...'
                sh 'npm install-test'
                  }
        }
        stage('Build Docker Image') {
             steps {
                script{
                    echo 'Building Docker image...'
                    dockerImage = docker.build imageName
                }
             }
        }
        stage('Deploy to Docker Hub') {
            steps {
               script {
                    echo 'Publishing Image to Docker Hub...'
                    docker.withRegistry( '', dockerHubCredential ) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
             }
        }
        stage('Cleanup Local Image') {
            steps {
               script {
                    sh "docker rmi $imageName:$BUILD_NUMBER"
                    sh "docker rmi $imageName:latest"
                    }
                }
        }
        stage('Provision Cluster & Deploy Image') {
            steps {
                script {
                    echo 'Provisioning Kubernetes Cluster...'
                    withCredentials([
			    [$class: 'UsernamePasswordMultiBinding', credentialsId: "${awsCredential}",
				        usernameVariable: 'DEPLOYMENT_USERNAME', passwordVariable: 'DEPLOYMENT_PASSWORD']
		     ]) {
                    
                    sh 'terraform init -backend-config=\"access_key=$DEPLOYMENT_USERNAME\"  -backend-config=\"secret_key=$DEPLOYMENT_PASSWORD\"'
                    sh 'terraform plan -out=plan.tfplan -var deployment_username=$DEPLOYMENT_USERNAME -var deployment_password=$DEPLOYMENT_PASSWORD'
		    sh 'terraform apply -auto-approve plan.tfplan'
	            app_url = sh (
			script: "terraform output app_url",
                        returnStdout: true
                     ).trim()   
                    }
                }
            }
        }
   
    }
    post { 
        success {
          
		slackSend channel:  "${slackChannel}",
          	color: COLOR_MAP[currentBuild.currentResult],
          	message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}\n APP_URL:${app_url}"
          
	  	sh "${successAction}"
        
        }
    
        failure {
	  	slackSend channel:  "${slackChannel}",
          	color: COLOR_MAP[currentBuild.currentResult],
          	message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}\n"
          
		sh "${failureAction}"
        
        }
    }
}
