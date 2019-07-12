#!/usr/bin/env groovy
import org.jenkinsci.plugins.workflow.steps.FlowInterruptedException
import hudson.AbortException

library identifier: 'jenkins-pipeline-library@gitlab', retriever: modernSCM(
  [$class: 'GitSCMSource',
   remote: 'git@github.com:bhargavb7/maven-project.git'])

podTemplate(label: 'master', containers: [
    containerTemplate(image: 'alpine/git', name: 'git', command: 'cat', ttyEnabled: true),
    containerTemplate(
        name: 'maven',
        image: 'maven:3.3.9-jdk-8-alpine',
        envVars: [
            envVar(key: 'MAVEN_SETTINGS_PATH', value: '/root/.m2/settings.xml'),
            secretEnvVar(key: 'GITLAB_TOKEN', secretName: 'git-secrets', secretKey: 'gitlab-access-token')
        ],
        ttyEnabled: true,
        command: 'cat'),
    containerTemplate(image: 'docker', name: 'docker', command: 'cat', ttyEnabled: true),
    containerTemplate(image: 'owasp/zap2docker-stable', name: 'owasp-zap', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.0', command: 'cat', ttyEnabled: true),
  ], volumes: [
    secretVolume(mountPath: '/root/.m2/', secretName: 'jenkins-maven-settings'),
    secretVolume(mountPath: '/home/jenkins/.docker', secretName: 'regsecret'),
    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
  ], imagePullSecrets: [ 'regsecret' ]) {

    node('mypod') {
        def jobName = "${env.JOB_NAME}".tokenize('/').last()
        def branchName = jobName
        def serviceName = "${env.JOB_NAME}".tokenize('/')[0]
        def projectNamespace = serviceName
        def DNS_NAME = "api.cicd.aagsiriuscom.com"
        def repositoryName = serviceName
        def baseJobUrl = "${env.JENKINS_URL}/blue/organizations/jenkins/${serviceName}/detail/${branchName}/${BUILD_NUMBER}"
        def buildUrl = "${baseJobUrl}/pipeline"
        def zapReportName = "ZAPScan"
        def zapReport = "${env.JENKINS_URL}/job/${serviceName}/job/${branchName}/${BUILD_NUMBER}/${zapReportName}"
        def testsUrl = "${baseJobUrl}/tests"
        def consoleUrl = "${env.JENKINS_URL}/job/${serviceName}/job/${branchName}/${BUILD_NUMBER}/console"
        

        rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${buildUrl}) - STARTED.", rawMessage: true
        checkout scm
        def featureBranch = false
        if (!branchName.equals("master")) {
            featureBranch = true
        }

        try {
            def accessToken = ""
            def commitHash = ""

            container('git') {
                stage('Get branch git comments') {
                    currentRevision = sh(returnStdout: true, script: "git rev-parse --verify HEAD | tr -d '\n'")
                    commitHash = sh(returnStdout: true, script: "git log --pretty=format:%H origin/master..${currentRevision} | tr '\n' ','").trim()
                }
            }

            if (!featureBranch) {
                container('kubectl') {
                    stage('Configure Kubernetes') {
                        sleep 30
                        createNamespace(projectNamespace)
                        sh """
                            kubectl label namespaces ${projectNamespace} kubesec-validation=enabled --overwrite=true
                        """
                    }
                }
            }

            lock('maven-build') { 
                container('maven') {
                    accessToken = sh(returnStdout: true, script: 'echo $GITLAB_TOKEN').trim()
                    stage('Build a project') {
                        sh 'mvn clean install -DskipTests=true'
                    }

                    stage('Run tests') {
                        try {
                            sh 'mvn clean install test'
                        } finally {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }

                    stage('SonarQube Analysis') {
                        if (!featureBranch) {
                            sonarQubeScanner(accessToken, "forsythe-aag-apps/${serviceName}", "https://sonarqube.${DNS_NAME}", branchName)
                        } else {
                            sonarQubeGitlabScanner(accessToken, "sirius-aag/dev-ops/cicd/${serviceName}", "https://sonarqube.${DNS_NAME}", branchName, commitHash)
                        }
                    }

                    if (!featureBranch) {
                        stage('Deploy project to Nexus') {
                            sh 'mvn -DskipTests=true package deploy'
                            archiveArtifacts artifacts: 'target/*.jar'
                        }
                    }
                }
            }

            if (!featureBranch) {
                container('docker') {
                    container('docker') {
                        stage('Docker build') {
                            sh "docker build -t ${serviceName} ."
                            sh "docker tag ${serviceName} registry.${DNS_NAME}/library/${repositoryName}"
                            sh "docker push registry.${DNS_NAME}/library/${repositoryName}"
                        }
                    }
                }

                container('kubectl') {
                    stage('Deploy MicroService') {
                       sh """
                           sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/deployment.yml | sed -e 's/{{REPOSITORY_NAME}}/'$repositoryName'/g' > ./deployment/deployment2.yml
                           sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/service.yml  > ./deployment/service2.yml
                           sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/prometheus-service-monitor.yml  > ./deployment/prometheus-service-monitor2.yml
                           sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/ingress.yml  > ./deployment/ingress2.yml

                           kubectl delete -f ./deployment/deployment2.yml -n ${projectNamespace} --ignore-not-found=true
                           kubectl delete -f ./deployment/service2.yml -n ${projectNamespace} --ignore-not-found=true
                           kubectl delete -f ./deployment/prometheus-service-monitor2.yml -n cicd-tools --ignore-not-found=true
                           kubectl delete -f ./deployment/ingress2.yml -n ${projectNamespace} --ignore-not-found=true

                           kubectl create -f ./deployment/deployment2.yml -n ${projectNamespace}
                           kubectl create -f ./deployment/service2.yml -n ${projectNamespace}
                           kubectl create -f ./deployment/prometheus-service-monitor2.yml -n cicd-tools
                           kubectl create -f ./deployment/ingress2.yml -n ${projectNamespace}
                       """

                       waitForRunningState(projectNamespace)
                       sleep 30
                       print "${serviceName} can be accessed at: http://${serviceName}.${DNS_NAME}  "
                       rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${buildUrl}) - DEPLOYED successfully at https://${serviceName}.${DNS_NAME}. Test result is [here](${testsUrl}).", rawMessage: true
                    }
                }
            }

            stage('ZAP testing'){
              def zaptestresult = ""

              if (!featureBranch) {
                zaptestresult = zapFullScan( serviceName, DNS_NAME)
              }else {
                zaptestresult = zapBaselineScan( serviceName, DNS_NAME)
              }

                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: '', reportFiles: 'zapreport.html', reportName: "${zapReportName}", reportTitles: 'OWASP-ZAP Scan'])

                rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${buildUrl})  ZAP test result : `${zaptestresult}`   [ZAP Scanning Report](${zapReport})", rawMessage: true
              }
              
            container('kubectl') {
                try {
                    rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${buildUrl}) - AWAITS for user input [here](${consoleUrl}).", rawMessage: true
                    timeout(time: 3, unit: 'MINUTES') {
                        input message: "Deploy to Production?"
                    }
                    container('kubectl') {
                        serviceName = "prod-${serviceName}"
                        projectNamespace = serviceName
                        sh """
                       kubectl create namespace ${projectNamespace} || true
                       kubectl label namespaces ${projectNamespace} kubesec-validation=enabled --overwrite=true
                       sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/deployment.yml | sed -e 's/{{REPOSITORY_NAME}}/'$repositoryName'/g' > ./deployment/deployment2.yml
                       sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/service.yml  > ./deployment/service2.yml
                       sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/prometheus-service-monitor.yml  > ./deployment/prometheus-service-monitor2.yml
                       sed -e 's/{{SERVICE_NAME}}/'$serviceName'/g' ./deployment/ingress.yml  > ./deployment/ingress2.yml

                       kubectl delete -f ./deployment/deployment2.yml -n ${projectNamespace} --ignore-not-found=true
                       kubectl delete -f ./deployment/service2.yml -n ${projectNamespace} --ignore-not-found=true
                       kubectl delete -f ./deployment/prometheus-service-monitor2.yml -n cicd-tools --ignore-not-found=true
                       kubectl delete -f ./deployment/ingress2.yml -n ${projectNamespace} --ignore-not-found=true

                       kubectl create -f ./deployment/deployment2.yml -n ${projectNamespace}
                       kubectl create -f ./deployment/service2.yml -n ${projectNamespace}
                       kubectl create -f ./deployment/prometheus-service-monitor2.yml -n cicd-tools
                       kubectl create -f ./deployment/ingress2.yml -n ${projectNamespace}
                   """

                        waitForRunningState(projectNamespace)
                        sleep 60
                        rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${buildUrl}) - DEPLOYED successfully at http://${serviceName}.${DNS_NAME}", rawMessage: true
                        print "${serviceName} can be accessed at: http://${serviceName}.${DNS_NAME}"
                    }
                } catch(err) {
                    def user = err.getCauses()[0].getUser().toString()
                    if('SYSTEM' == user) { 
                      rocketSend channel: 'jenkins', message: "@here prod-${serviceName} build [#${BUILD_NUMBER}](${buildUrl}) - TIMED OUT after 3 minutes.", rawMessage: true
                    } else {
                      rocketSend channel: 'jenkins', message: "@here prod-${serviceName} build [#${BUILD_NUMBER}](${buildUrl}) - ABORTED by ${user}", rawMessage: true
                    }
                }
            }
        }catch (FlowInterruptedException e){
            echo 'Cause(s) of Interruption : '
            e.getCauses().eachWithIndex{ cause ->
             println  " ${cause.getShortDescription()}"
            }

            currentBuild.result = 'ABORTED'
            rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${consoleUrl}) - ABORTED manually.", rawMessage: true
        } 

        catch(AbortException e){
          echo 'Err: Failed with Error: ' + e.toString()
          currentBuild.result = 'ABORTED'
          rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${consoleUrl}) - ABORTED manually.", rawMessage: true
        }

        catch (e) {
          currentBuild.result = 'FAILURE'
              rocketSend channel: 'jenkins', message: "@here ${serviceName} build [#${BUILD_NUMBER}](${consoleUrl}) - FAILED.", rawMessage: true

            echo 'Err: Incremental Build failed with Error: ' + e.toString()
 
            
        }
    }
}
