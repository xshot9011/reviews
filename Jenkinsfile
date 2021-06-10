def scmVars

pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:20.10.3-dind
    command:
    - dockerd
    - --host=unix:///var/run/docker.sock
    - --host=tcp://0.0.0.0:2375
    - --storage-driver=overlay2
    tty: true
    securityContext:
        privileged: true
  - name: helm
    image: lachlanevenson/k8s-helm:v3.5.0
    command:
    - cat
    tty: true
  - name: skan
    image: alcide/skan:v0.9.0-debug
    command:
    - cat
    tty: true
  - name: gradle
    image: gradle:6.0.1-jdk11 
    command:
    - cat
    tty: true
    securityContext:
        privileged: true
    volumeMounts:
    - mountPath: '/home/jenkins/dependency-check-data'
      name: dependency-check-data
  volumes:
  - name: dependency-check-data
    hostPath:
      path: /tmp/dependency-check-data
"""
        }
    }
    
	environment {
        ENV_NAME = "${BRANCH_NAME == "master" ? "uat" : "${BRANCH_NAME}"}"
        SCANNER_HOME = tool 'sonarqube-scanner'
        PROJECT_KEY = 'bookinfo-reviews'
        PROJECT_NAME = 'bookinfo-reviews'
    }
	
    stages {
        
        stage('Clone ratings source code') {
            steps {
                container('jnlp') {
                    script {
                        scmVars = git branch: "${BRANCH_NAME}",
                                      credentialsId: 'bookinfo-git-deploy-key',
                                      url: 'git@gitlab.com:workshop35/bookinfo/reviews.git'
                    }
                }
            }
        }

        stage('sKan') {
            steps{
                container('helm') {
                    script {
                        sh "helm template -f k8s/helm-values/values-bookinfo-${ENV_NAME}-reviews.yaml \
                             --set extraEnv.COMMIT_ID=${scmVars.GIT_COMMIT} \
                             --namespace opsta-${ENV_NAME} bookinfo-${ENV_NAME}-reviews k8s/helm \
                             > k8s-minifest-deploy.yaml"
                    }
                }
                container('skan') {
                    script {
                        sh "/skan manifest -f k8s-minifest-deploy.yaml"
                        archiveArtifacts artifacts: 'skan-result.html'
                        sh "rm k8s-minifest-deploy.yaml"
                    }
                }
            }
        }
        
        
        stage('Sonarqube Scanner') {
            steps {
                container('gradle') {
                    script {
                        withSonarQubeEnv('sonarqube-server') {
                            sh '''gradle sonarqube \
                                    -Dsonar.projectKey=${PROJECT_KEY} \
                                    -Dsonar.projectName=${PROJECT_NAME} \
                                    -Dsonar.ProjectVersion=${BRANCH_NAME}-${BUILD_NUMBER} \
                                    -Dsonar.sources=./src
                            '''
                        }

                        timeout(time: 1, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }
        
        stage('OWASP dependency check') {
            steps {
                container('gradle') {
                    script {
                        dependencyCheck(
                            additionalArguments: "--data /home/jenkins/dependency-check-data --out dependency-check-report.xml",
                            odcInstallation: "dependency-check" // plugin
                        )

                        dependencyCheckPublisher(
                            pattern: 'dependency-check-report.xml'
                        )
                    }
                }
            }
        }
		
		stage('Build ratings Docker Image and push') {
            steps {
                container('docker') {
                    script {
                        docker.withRegistry('', 'registry-bookinfo') {
                            docker.build('bigdev2000/reviews:${ENV_NAME}').push()
                        }
                    }
                }
            }
        }

        stage('Anchore Engine') {
            steps {
                container('jnlp') {
                    script {
                        writeFile file: 'anchore_images', text: "bigdev2000/reviews:${ENV_NAME}"
                        anchore name: 'anchore_images', bailOnFail: false  // if fail just pass, something we cannot fix
                    }
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                container('helm') {
                    script {
						withKubeConfig([credentialsId: 'gke-opsta-cluster-sa-secret-file']) {
							sh "helm upgrade -f k8s/helm-values/values-bookinfo-${ENV_NAME}-reviews.yaml --wait \
                                 --set extraEnv.COMMIT_ID=${scmVars.GIT_COMMIT} \
                                 --namespace opsta-${ENV_NAME} bookinfo-${ENV_NAME}-reviews k8s/helm/ \
                                 --install"
						}
                    }
                }
            }
        }
    }
}
