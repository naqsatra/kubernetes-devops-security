pipeline {
  agent any
  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "naqsatra/numeric-app:${GIT_COMMIT}"
    applicationURL="http://devsecops-dev.eastus.cloudapp.azure.com"
    applicationURI="/increment/99"
  }
  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later
            }
        } 
    //     stage('SonarQube - SAST') {
    //   steps {
    //     withSonarQubeEnv('SonarQube') {
    //       sh "mvn sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.host.url=http://devsecops-dev.eastus.cloudapp.azure.com:9000"
    //     }
    //     timeout(time: 2, unit: 'MINUTES') {
    //       script {
    //         waitForQualityGate abortPipeline: true
    //       }
    //     }
    //   }
    // }
    stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
        	"Dependency Scan": {
        		sh "mvn dependency-check:check"
			},
			"Trivy Scan":{
				sh "bash trivy-docker-image-scan.sh"
			},
			"OPA Conftest":{
				sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
			}   	
      	)
      }
    }
    
        stage('Docker Build and Push') {
            steps {
              withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
                sh 'printenv'
                sh 'sudo docker build -t naqsatra/numeric-app:""$GIT_COMMIT"" .'
                sh 'docker push naqsatra/numeric-app:""$GIT_COMMIT""'
              }
            }
          } 
    stage('K8S Deployment - DEV') {
      steps {
        parallel(
          "Deployment": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment.sh"
            }
          },
          "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment-rollout-status.sh"
            }
          }
        )
      }
    }
    //     stage('Integration Tests - DEV') {
    //   steps {
    //     script {
    //       try {
    //         withKubeConfig([credentialsId: 'kubeconfig']) {
    //          // sh "kubectl -n default get pods"
    //           sh "bash integration-test.sh"
    //         }
    //       } catch (e) {
    //         withKubeConfig([credentialsId: 'kubeconfig']) {
    //           sh "kubectl -n default rollout undo deploy ${deploymentName}"
    //         }
    //         throw e
    //       }
    //     }
    //   }
    // }     
    }
}