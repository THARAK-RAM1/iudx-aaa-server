properties([pipelineTriggers([githubPush()])])
pipeline {
  environment {
    devRegistry = 'ghcr.io/karun-singh/aaa-dev'
    deplRegistry = 'ghcr.io/karun-singh/aaa-depl'
    testRegistry = 'ghcr.io/karun-singh/aaa-test:latest'
    registryUri = 'https://ghcr.io'
    registryCredential = 'karun-ghcr'
    GIT_HASH = GIT_COMMIT.take(7)
  }
  agent { 
    node {
      label 'slave1' 
    }
  }
  stages {

    stage('Building images') {
      steps{
        script {
          echo 'Pulled - ' + env.GIT_BRANCH
          devImage = docker.build( devRegistry, "-f ./docker/dev.dockerfile .")
          deplImage = docker.build( deplRegistry, "-f ./docker/depl.dockerfile .")
          testImage = docker.build( testRegistry, "-f ./docker/test.dockerfile .")
        }
      }
    }

    stage('Run Unit Tests and CodeCoverage test'){
      steps{
        script{
          sh 'docker-compose -f docker-compose-test.yml up test'
        }
      }
    }

    stage('Capture Unit Test results'){
      steps{
        xunit (
          thresholds: [ skipped(failureThreshold: '0'), failed(failureThreshold: '10') ],
          tools: [ JUnit(pattern: 'target/surefire-reports/*.xml') ]
        )
      }
      post{
        failure{
          error "Test failure. Stopping pipeline execution!"
        }
      }
    }

    stage('Capture Code Coverage'){
      steps{
        jacoco classPattern: 'target/classes', execPattern: 'target/jacoco.exec', sourcePattern: 'src/main/java'
      }
    }

    stage('Run aaa-server for Integration Test'){
      steps{
        script{
            sh 'docker/runIntegTests.sh'
            sh 'scp src/test/resources/Integration_Test.postman_collection.json jenkins@jenkins-master:/var/lib/jenkins/iudx/aaa/Newman/'
            sh 'docker-compose -f docker-compose-test.yml up -d integTest'
            sh 'sleep 45'
        }
      }
      post{
        failure{
          script{
            sh 'mvn flyway:clean -Dflyway.configFiles=/home/ubuntu/configs/aaa-flyway.conf'
            cleanWs(patterns:[[pattern:'./src/main/resources/db/migration/*',type:'INCLUDE']])
          }
        }
      }
    }

    stage('Integration Test & OWASP ZAP pen test'){
      steps{
        node('master') {
          script{
            startZap ([host: 'localhost', port: 8090, zapHome: '/var/lib/jenkins/tools/com.cloudbees.jenkins.plugins.customtools.CustomTool/OWASP_ZAP/ZAP_2.11.0'])
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              sh 'HTTP_PROXY=\'127.0.0.1:8090\' newman run /var/lib/jenkins/iudx/aaa/Newman/Integration_Test.postman_collection.json -e /home/ubuntu/configs/aaa-postman-env.json --insecure -r htmlextra --reporter-htmlextra-export /var/lib/jenkins/iudx/aaa/Newman/report/report.html'
            }
            runZapAttack()
          }
        }
      }
      post{
        always{
          node('master') {
            script{
              archiveZap failAllAlerts: 15
              publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '/var/lib/jenkins/iudx/aaa/Newman/report/', reportFiles: 'report.html', reportName: 'HTML Report', reportTitles: '', reportName: 'Integration Test Report'])
            }  
          }
          script{
            sh 'mvn flyway:clean -Dflyway.configFiles=/home/ubuntu/configs/aaa-flyway.conf'
            sh 'docker-compose -f docker-compose-test.yml logs integTest > aaa.log'
            sh 'scp aaa.log jenkins@jenkins-master:/var/lib/jenkins/userContent/'
            echo 'container logs (aaa.log) can be found at jenkins-url/userContent'
            sh 'docker-compose -f docker-compose-test.yml down --remove-orphans'
            cleanWs(patterns:[[pattern:'./src/main/resources/db/migration/*',type:'INCLUDE']])
          }
        }
      }
    }

    stage('Push Image') {
      when{
        expression {
          return env.GIT_BRANCH == 'origin/main';
        }
      }
      steps{
        script {
          docker.withRegistry( registryUri, registryCredential ) {
            devImage.push("3.0-${env.GIT_HASH}")
            deplImage.push("3.0-${env.GIT_HASH}")
          }
        }
      }
    }
  }
}