pipeline {
    agent any
    environment {
        GCP_PROJECT = 'my-project-nexea'
        GCP_INSTANCE = 'nexea-event-app' // Ensure this is the external IP or hostname of your instance
        DOCKER_IMAGE = 'nexea-event-app'
        GCP_SA_KEY_FILE = credentials('gcp-sa-key') // Jenkins secret with your GCP service account key content
        SSH_CRED_ID = 'GCP_SSH_CREDENTIAL_ID' // SSH credential ID for GCP instance
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $DOCKER_IMAGE .'
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_SA_KEY_FILE')]) {
                        sh """
                        cat $GCP_SA_KEY_FILE | docker login -u _json_key --password-stdin https://gcr.io
                        docker tag $DOCKER_IMAGE gcr.io/$GCP_PROJECT/$DOCKER_IMAGE:latest
                        docker push gcr.io/$GCP_PROJECT/$DOCKER_IMAGE:latest
                        """
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                sshagent([SSH_CRED_ID]) {
                    script {
                        // Copy docker-compose.yml and nginx.conf to the GCP instance
                        sh '''
                        scp -o StrictHostKeyChecking=no docker-compose.yml nginx/nginx.conf famintech@$GCP_INSTANCE:~/
                        
                        # SSH into the GCP instance to run docker-compose commands
                        ssh -o StrictHostKeyChecking=no famintech@$GCP_INSTANCE "
                        docker pull gcr.io/$GCP_PROJECT/$DOCKER_IMAGE:latest &&
                        docker-compose -f ~/docker-compose.yml down &&
                        docker-compose -f ~/docker-compose.yml up -d"
                        '''
                    }
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
