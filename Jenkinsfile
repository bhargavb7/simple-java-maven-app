node('master'){

    stage('git checkout'){
    try {    
        checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[ url: 'git@github.com:bhargavb7/simple-java-maven-app.git']]])
    } catch (e) {
            currentBuild.result = "FAILED"
            emailNotify(currentBuild.result, env.BUILD_USER)
            throw e
            }
    }

}

