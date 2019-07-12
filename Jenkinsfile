pipeline {
    agent any
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