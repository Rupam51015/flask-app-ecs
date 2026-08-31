pipeline{
    agent any;
    stages{
        stage("code clone"){
            steps{
                git url: "https://github.com/Rupam51015/flask-app-ecs.git", branch: "main"
            }
        }
        stage("Build image"){
            steps{
                sh "docker build --no-cache -f Dockerfile-multi -t flask-app ."
            }
        }
        stage("Push to dockerHub"){
            steps{
                withCredentials([usernamePassword(
                    credentialsId:"dockerHubCreds",
                    passwordVariable:"dockerHubPass",
                    usernameVariable:"dockerHubUser"
                    )]){
                        sh "docker login -u ${dockerHubUser} -p ${dockerHubPass}"
                        sh "docker image tag flask-app ${dockerHubUser}/flask-app"
                        sh "docker push ${dockerHubUser}/flask-app:latest"
                    }
            }
        }
        stage("Deploy"){
            steps{
                sh "docker compose up -d"
            }
        }
        stage("Healthcheck"){
            steps{
                sh '''
                    timeout 30s bash -c '
                    until [ "$(docker compose ps flask-app --format "{{.Health}}")" == "healthy" ]; do
                        echo "Waiting for Flask app to pass health check..."
                        sleep 2
                    done'
                '''
            }
        }
    }
    post{
        failure {
            script {
                emailext from: 'rumapdebnath75@gmail.com',
                    body: "Build Failed",
                    subject: "Bad News: Your build has failed!",
                    to: 'rumapdebnath75@gmail.com'
            }
        }
    }
}
