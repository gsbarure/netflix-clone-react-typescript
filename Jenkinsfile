pipeline{
    agent any
    tools{
        jdk 'jdk'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME=tool 'scanner'
    }
    stages {
        stage('1.0 clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('2.0 Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/FortressTechnologiesInc/netflzapp.git'
            }
        }
       stage('3.0 OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("4.0 Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=netflzapp -Dsonar.java.binaries=. -Dsonar.projectKey=netflzapp '''
                }
            }
        }
        stage('Quality Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar' 
                }
            }
        }
        stage("5.0 Trivy FS SCAN") {
            steps {
                sh "trivy fs ."
            }
        }
        stage("6.0 Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=36a50cd7bedc727a9c31ebf3d41fc568 -t netflzapp . -f /root/appnode/workspace/netflzapp/Dockerfile"
                       sh "docker tag netflzapp limkel/netflzapp "
                       sh "docker push limkel/netflzapp "
                       sh "trivy image limkel/netflzapp > trivyimage.txt"
                       sh "docker rmi -f limkel/netflzapp "
                    }
                }
            }
        }
        stage("7.0 TRIVY"){
            steps{
                sh "trivy image limkel/netflzapp > trivyimage.txt" 
            }
        }
        
        stage('8.0 Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'spjt', namespace: 'netflzapp', restrictKubeConfigAccess: false, serverUrl: 'https://192.168.49.2:8443') {
                                sh 'kubectl apply -f /root/appnode/workspace/netflzapp/deployment.yml -n netflzapp'
                                sh 'kubectl get po -n netflzapp'
                                sh 'kubectl get svc  -n netflzapp'
                        }   
                    }
                }
            }
        }
        stage('9.0clean workspace'){
            steps{
                cleanWs()
            }
        }

    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'deniferdavies@gmail.com', 
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
        
}
