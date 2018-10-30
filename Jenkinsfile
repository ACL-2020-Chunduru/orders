@Library('dynatrace@master') _

pipeline {
  agent {
    label 'maven'
  }
  environment {
    APP_NAME = "orders"
    ARTEFACT_ID = "sockshop/" + "${env.APP_NAME}"
    VERSION = readFile 'version'
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/library/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}-${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}-${env.VERSION}"
  }
  stages {
    stage('Maven build') {
      steps {
        checkout scm
        container('maven') {
          sh 'mvn -B clean package'
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker build -t ${env.TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker push ${env.TAG_DEV}"
        }
      }
    }
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "sed -i 's#image: .*#image: ${env.TAG_DEV}#' manifest/orders.yml"
          sh "kubectl -n dev apply -f manifest/orders.yml"
        }
      }
    }
    stage('Run health check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "waiting for the service to start..."
        sleep 100

        sh "rm -rf test_results"
        sh "mkdir test_results"

        container('jmeter') {
          executeJMeter ( 
            scriptName: 'jmeter/basiccheck.jmx', 
            serverUrl: "${env.APP_NAME}.dev", 
            serverPort: 80,
            checkPath: '/health',
            vuCount: 1,
            loopCount: 1,
            LTN: "HealthCheck_${BUILD_NUMBER}",
            funcValidation: true,
            avgRtValidation: 0
          )
        }
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        sh "rm -rf test_results"
        sh "mkdir test_results"

        container('jmeter') {
          executeJMeter ( 
            scriptName: "jmeter/${env.APP_NAME}_load.jmx", 
            serverUrl: "${env.APP_NAME}.dev", 
            serverPort: 80,
            checkPath: '/health',
            vuCount: 1,
            loopCount: 1,
            LTN: "FuncCheck_${BUILD_NUMBER}",
            funcValidation: true,
            avgRtValidation: 0
          )
        }
      }
    }
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
          sh "docker push ${env.TAG_STAGING}"
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${env.TAG_STAGING}"),
            string(name: 'VERSION', value: "${env.VERSION}")
          ]
      }
    }
  }
}
