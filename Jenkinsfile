pipeline {
    agent any
    
    environment{
        SONAR_HOME = tool 'sonar'
        DOCKER_CRED = credentials('dockercred')
        BUILD_NAME = "v${BUILD_NUMBER}"
        IMAGE_NAME = "megabankapp"
    }
    tools{
        maven 'maven3'
    }
    
    stages {
        stage('Clone the Repo') {
            steps {
                git branch: 'main' , url: 'https://github.com/omnagare9975/Multi-Tier-BankApp-CI.git'
            }
        }
        stage('Maven Compile'){
            steps{
                sh 'mvn compile'
            }
        }
        stage('Maven Test'){
            steps{
                sh 'mvn test'
            }
        }

        stage('sonarqube anaylysis'){
            steps{
              withSonarQubeEnv('sonar-server'){
                     sh '$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=bank -Dsonar.projectKey=bank -Dsonar.java.binaries=target'
                }
              timeout(time: 1 , unit: 'MINUTES'){
                  waitForQualityGate abortPipeline: false
              }
           }
        }
        stage('trivy scan'){
            steps{
                sh 'trivy fs --format table -o trivy-scan.html .'
            }
        }
         stage('Artifact Generate'){
            steps{
                sh 'mvn clean package '
            }
        }
        
        stage('Deploy the Artifact'){
            steps{
                withMaven(globalMavenSettingsConfig: 'mavenpom', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy'
              }
            }
        }
        
        stage('Docker Build And Push'){
            steps{
                sh 'docker build -t $DOCKER_CRED_USR/$IMAGE_NAME:$BUILD_NAME .'
            }
        }
        stage('Trivy Scan The Image'){
            steps{
                sh 'trivy image --format table -o trivy-image-report.html $DOCKER_CRED_USR/$IMAGE_NAME:$BUILD_NAME'
            }
        }
        stage('Push The Docker Image'){
            steps{
               sh 'echo "$DOCKER_CRED_PSW" | docker login -u $DOCKER_CRED_USR --password-stdin'
                 
               sh 'docker push $DOCKER_CRED_USR/$IMAGE_NAME:$BUILD_NAME'
            }
        }
        stage('Update Manifest File in Bank APplication CD') {
              steps {
                    script {
                       cleanWs()

                        withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                         sh '''
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/omnagare9975/Multi-Tier-BankApp-CD.git
                            cd Multi-Tier-BankApp-CD
                            sed -i "s|omnagare/megabankapp:.*|omnagare/megabankapp:${BUILD_NAME}|" k8s/bankapp-ds.yml

                   
                            echo "Updated manifest file contents:"
                            cat  k8s/bankapp-ds.yml

                    
                           git config user.name "omnagare"
                           git config user.email "omnagare07@gmail.com"
                           git add k8s/bankapp-ds.yml
                           git commit -m "Update image tag to ${BUILD_NAME}"
                           git push origin main
                           '''
                   }
                }
            }
         }

    }
    
post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? '#4CAF50' : '#F44336'

            def body = """
                <html>
                <body style="font-family: Arial, sans-serif;">
                    <div style="border: 2px solid ${bannerColor}; padding: 20px; border-radius: 10px;">
                        <h2 style="color: ${bannerColor};">
                            🔧 Job: ${jobName} | Build #${buildNumber}
                        </h2>
                        <div style="background-color: ${bannerColor}; color: white; padding: 10px; border-radius: 5px;">
                            <h3 style="margin: 0;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                        </div>
                        <p style="margin-top: 20px;">
                            📄 View full logs: <a href="${BUILD_URL}" style="color: #1E88E5;">Console Output</a>
                        </p>
                    </div>
                    <p style="font-size: 12px; color: gray; margin-top: 10px;">
                        This is an automated message from Om Nagare.
                    </p>
                </body>
                </html>
            """

            emailext(
                subject: "📣 ${jobName} - Build #${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                mimeType: 'text/html',
                to: 'omnagare83@gmail.com',
                from: 'omnagare07@gmail.com',
                replyTo: 'omnagare07@gmail.com'
            )
        }
    }
}

    
}
