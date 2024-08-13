pipeline {
    triggers {
      pollSCM('')
    }  
    agent any
    environment{
        GIT_REPO_NAME = 'argocd-devops'
    }
    stages {        
        stage('Run Automation Test Cases') {
          agent {
              docker { image 'python:3' }
          }          
            steps {
                script {
                    sh "pip3 install -r requirements.txt"
                    sh "python3 test.py"
                }
            }
            post {
              always {
                junit 'test-reports/*.xml'
              }
            }    
        }

        stage('Docker Build') {
            steps {
                script {
                  sh "docker build -t ${GIT_COMMIT} ."
                  sh "docker tag ${GIT_COMMIT}:latest chaudharishubham2911/argocd-demo:${GIT_COMMIT}"
                }
            }
        }

        stage('Docker Image Scanning') {
            steps {
                script {
                  sh "curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl"
                  sh "trivy image --format template --template '@html.tpl' --output trivy_report.html --exit-code 0 --severity HIGH,CRITICAL chaudharishubham2911/argocd-demo:${GIT_COMMIT}"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "trivy_report.html", fingerprint: true      
                    publishHTML (target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'trivy_report.html',
                        reportName: 'Trivy Scan',
                        ])
                    }
             }            
        }

         stage('Docker Push') {
             steps {
                 script {
                         def registryCredentials = [
                         credentialsId: 'docker-creds'
                         ]
                         withCredentials([usernamePassword(credentialsId: 'docker-creds', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                         sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPassword}"
                         sh "docker push chaudharishubham2911/argocd-demo:${GIT_COMMIT}"
                     }
                 }
             }
         }

         stage('Update Helm Values File') {
             steps {
                 script {
                        withCredentials([usernamePassword(credentialsId: 'git-creds', passwordVariable: 'GitPassword', usernameVariable: 'GitUser')]) {
                            sh '''
                                git clone https://${GitPassword}@github.com/${GitUser}/${GIT_REPO_NAME}
                                cd ${WORKSPACE}/${GIT_REPO_NAME}
                                sed -i "s/IMAGE_ID/${GIT_COMMIT}/g" helm-chart/values.yaml
                                git add helm-chart/values.yaml
                                git commit -m "Update image version to ${GIT_COMMIT}"
                                git push https://${GitPassword}@github.com/${GitUser}/${GIT_REPO_NAME} HEAD:main
                            '''
                        stash name: "${GIT_REPO_NAME}", includes: '**/*', path: "${GIT_REPO_NAME}"   
                        }
                     }
                 }
            }

         stage('Create ArgoCD Application') {
           when {
             expression {
               def CREATE_APP = env.CREATE_APP
               return CREATE_APP == 'Yes'
             }
            }          
             steps {
                 script {
                        unstash "${GIT_REPO_NAME}"
                            sh '''  
                                cd ${WORKSPACE}/${GIT_REPO_NAME}
                                kubectl apply -f application.yaml --kubeconfig ${WORKSPACE}/kubeconfig
                            '''
                     }
                 }
            }            
    }

        post {
          always {
            cleanWs()
          }
        }
}
