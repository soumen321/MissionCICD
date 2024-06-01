pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
              git branch: 'main', changelog: false, poll: false, url: 'https://github.com/soumen321/MissionCICD.git'  
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
         stage('Testing') {
            steps {
               sh 'mvn test -DskipTests=true' 
            }
        }
        
        stage('Trivy Scan File System') {
            steps {
                     sh "trivy fs --format table -o trivy-fs-report.html ."
                }
         }
         stage('SonarQube Analysis') {
            steps {
               withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=MissionCICD -Dsonar.projectName=MissionCICD \
                            -Dsonar.java.binaries=. '''

                }
            }
        }
         stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }
        stage('Deploy Artifacts to Nexus') {
            steps {
               withMaven(globalMavenSettingsConfig: 'maven-setting', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }
      
        stage('Build docker image'){
            steps{
                script{
                      sh 'docker build -t techsoumen/missioncicd:latest .'
                }
            }
        }
        
        stage('Trivy Scan Image') {
            steps {
                     sh "trivy image --format table -o trivy-image-report.html techsoumen/missioncicd:latest"
                }
         }
          stage('Push to Docker Hub'){
            steps {
                echo "Pushing the image to docker image"
                 script {
                    
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                       sh "docker tag techsoumen/missioncicd techsoumen/missioncicd:latest"    
                       sh 'docker push techsoumen/missioncicd:latest'
                   }
                    
                 }
               
            } 
        }
        //stage('Deploy to Container'){
           // steps{
               // script{
                   // withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                       // sh 'docker run -d -p 8083:8080 techsoumen/missioncicd:latest'
                    //}
               // }
           // }
       // }
         stage('Deploy to k8s') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'DS-EKS', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://0819BEF7F0F5B2DAF990CB86576BC3C8.yl4.us-east-1.eks.amazonaws.com') {
                    sh 'kubectl apply -f ds.yml -n webapps'
                    sleep 60
                }
             }
         }
         stage('Verify Deployment'){
             steps {
                withKubeConfig(caCertificate: '', clusterName: 'DS-EKS', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://0819BEF7F0F5B2DAF990CB86576BC3C8.yl4.us-east-1.eks.amazonaws.com') {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
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
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' 
? 'green' : 'red' 
 
                def body = """ 
                    <html> 
                    <body> 
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;"> 
                    <h2>${jobName} - Build ${buildNumber}</h2> 
                    <div style="background-color: ${bannerColor}; padding: 10px;"> 
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3> 
                    </div> 
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p> 
                    </div> 
                    </body> 
                    </html> 
                """ 
 
                emailext ( 
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}", 
                    body: body, 
                    to: '<to email>', 
                    from: 'jenkins@example.com', 
                    replyTo: 'jenkins@example.com', 
                    mimeType: 'text/html', 
                    attachmentsPattern: 'trivy-image-report.html' 
                ) 
            } 
        } 
    }
}