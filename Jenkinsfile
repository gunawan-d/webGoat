pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('DockerLogin')
        SNYK_TOKEN = credentials('snyk-api-token')
        // SONARQUBE_CREDENTIALS = credentials('SonarToken')
        SONARQUBE_CREDENTIALS_PSW = credentials('SONARQUBE_CREDENTIALS_PSW')
    }
    stages {
    	stage('Secret Scanning Using Trufflehog') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '-u root --entrypoint='
                }
            }
            steps {
                sh 'trufflehog filesystem . --exclude-paths trufflehog-excluded-paths.txt --json > trufflehog-scan-result.json'
                sh 'cat trufflehog-scan-result.json'
                archiveArtifacts artifacts: 'trufflehog-scan-result.json'
            }
        }
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9.4-eclipse-temurin-17-alpine'
                    args '-u root'
                }
            }
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        // stage('Test') {
        //     agent {
        //         docker {
        //             image 'maven:3.9.4-eclipse-temurin-17-alpine'
        //             args '-u root'
        //         }
        //     }
        //     steps {
        //         sh 'mvn test -Pcoverage'
        //     }
        //     post {
        //         always {
        //             junit 'target/surefire-reports/*.xml'
        //         }
        //     }
        // }
        stage('SCA Snyk Test') {
            agent {
              docker {
                  image 'snyk/snyk:node'
                  args '-u root --network host --env SNYK_TOKEN=$SNYK_CREDENTIALS_PSW --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'snyk test --json > snyk-scan-report.json'
                }
                sh 'cat snyk-scan-report.json'
                archiveArtifacts artifacts: 'snyk-scan-report.json'
            }
        }
        stage('SCA Trivy Scan Dockerfile Misconfiguration') {
            agent {
              docker {
                  image 'aquasec/trivy:latest'
                  args '-u root --network host --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trivy config Dockerfile --exit-code=1 --format json > trivy-scan-dockerfile-report.json'
                }
                sh 'cat trivy-scan-dockerfile-report.json'
                archiveArtifacts artifacts: 'trivy-scan-dockerfile-report.json'
            }
        }
        // stage('SAST Snyk') {
        //     agent {
        //       docker {
        //           image 'snyk/snyk:node'
        //           args '-u root --network host --env SNYK_TOKEN=$SNYK_CREDENTIALS_PSW --entrypoint='
        //       }
        //     }
        //     steps {
        //         catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        //             sh 'snyk code test --json > snyk-sast-report.json'
        //         }
        //         archiveArtifacts artifacts: 'snyk-sast-report.json'
        //     }
        // }
        stage('SAST SonarQube') {
            agent {
              docker {
                    image 'maven:3.9.4-eclipse-temurin-17-alpine'
                    args '-u root --network host'
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'mvn sonar:sonar -Dsonar.token=$SONARQUBE_CREDENTIALS_PSW -Dsonar.projectKey=WebGoat -Dsonar.qualitygate.wait=true -Dsonar.host.url=http://147.139.166.250:9009 -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco-unit-test-coverage-report/jacoco.xml' 
                }
            }
        }
        stage('Build Docker Image and Push to Docker Registry') {
            agent {
                docker {
                    image 'docker:dind'
                    args '--user root --network host -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker build -t gunawand/webgoat:0.1 .'
                sh 'docker push gunawand/webgoat:0.1'
            }
        }
        stage('Push Docker Image To CR') {
            agent {
                docker {
                    image 'docker:dind'
                    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker push gunawand/webgoat:0.1'
            }
        }
        // stage('DAST Nuclei') {
        //     agent {
        //         docker {
        //             image 'projectdiscovery/nuclei'
        //             args '--user root --network host --entrypoint='
        //         }
        //     }
        //     steps {
        //         catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        //             sh 'nuclei -u http://119.81.54.27:8080 -nc -j > nuclei-report.json'
        //             sh 'cat nuclei-report.json'
        //         }
        //         archiveArtifacts artifacts: 'nuclei-report.json'
        //     }
        // }
        // stage('DAST OWASP ZAP') {
        //     agent {
        //         docker {
        //             image 'ghcr.io/zaproxy/zaproxy:weekly'
        //             args '-u root --network host -v /var/run/docker.sock:/var/run/docker.sock --entrypoint= -v .:/zap/wrk/:rw'
        //         }
        //     }
        //     steps {
        //         catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        //             sh 'zap-baseline.py -t http://119.81.54.27:8080 -r zapbaseline.html -x zapbaseline.xml'
        //         }
        //         sh 'cp /zap/wrk/zapbaseline.html ./zapbaseline.html'
        //         sh 'cp /zap/wrk/zapbaseline.xml ./zapbaseline.xml'
        //         archiveArtifacts artifacts: 'zapbaseline.html'
        //         archiveArtifacts artifacts: 'zapbaseline.xml'
        //     }
        // }
    }
}
