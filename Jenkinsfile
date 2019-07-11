pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "willbla/train-schedule"
    }
    stages {
        stage('DeployApp') {
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'bb20015a-575d-475f-b78e-65ecd648c9fc',
                    configs: 'deployment.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}