pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
    skipDefaultCheckout(true)
  }

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    booleanParam(name: 'RUN_INTEGRATION_TESTS', defaultValue: true, description: 'Run integration tests?')
    booleanParam(name: 'DOCKER_PUSH', defaultValue: false, description: 'Push Docker image to registry after build?')
  }

  environment {
    APP_NAME = 'my-app'
    MAVEN_OPTS = '-Xms256m -Xmx1024m'
    DOCKER_IMAGE = "${env.APP_NAME}:${env.BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        echo "Checking out branch ${params.BRANCH}"
        checkout([
          $class: 'GitSCM',
          branches: [[name: "*/${params.BRANCH}"]],
          userRemoteConfigs: [[url: 'https://github.com/Herit1974/jenkin-pipeline-1.git']]
        ])
      }
    }

    stage('Prepare') {
      steps {
        script {
          isMaven = fileExists('pom.xml')
          isNode  = fileExists('package.json')
          hasDockerfile = fileExists('Dockerfile')
          echo "Detected: Maven=${isMaven}, Node=${isNode}, Dockerfile=${hasDockerfile}"
        }
      }
    }

    stage('Lint & Static Checks') {
      parallel {
        stage('Maven/Java Lint') {
          when { expression { return isMaven } }
          steps {
            sh 'mvn -B -DskipTests clean verify'
          }
        }

        stage('Node Lint') {
          when { expression { return isNode } }
          steps {
            sh 'npm ci || true'
            sh 'if [ -f package.json ]; then npm run lint || echo "no-lint-script"; fi'
          }
        }
      }
    }

    stage('Build') {
      steps {
        script {
          if (isMaven) {
            sh 'mvn -B -DskipTests package'
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
          } else if (isNode) {
            sh 'npm ci'
            sh 'npm run build || true'
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
          } else {
            sh 'echo "no-build" > NO_BUILD.txt'
            archiveArtifacts artifacts: 'NO_BUILD.txt'
          }
        }
      }
    }

    stage('Unit Tests') {
      steps {
        script {
          if (isMaven) {
            sh 'mvn -B test'
            junit '**/target/surefire-reports/*.xml'
          } else if (isNode) {
            sh 'npm test || true'
            junit 'test-results/*.xml'
          } else {
            echo 'No unit tests to run'
          }
        }
      }
      post {
        always {
          echo "Unit tests stage finished"
        }
      }
    }

    stage('Integration Tests (parallel)') {
      when { expression { return params.RUN_INTEGRATION_TESTS } }
      parallel {

        stage('Integration: DB (profile)') {
          steps {
            script {
              if (isMaven) {
                sh 'mvn -B -Pintegration verify || true'
                junit '**/target/failsafe-reports/*.xml'
              } else {
                echo 'No integration tests for non-Maven project'
              }
            }
          }
        }

        stage('Integration: API Endpoints') {
          steps {
            script {
              sh 'echo "Pretend: starting API tests"'
            }
          }
        }

      }
    }

    stage('Static Analysis / Sonar (optional)') {
      when { expression { return env.SONAR_HOST_URL != null } }
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh 'mvn -B sonar:sonar -Dsonar.login=${SONAR_TOKEN} -Dsonar.host.url=${SONAR_HOST_URL}'
        }
      }
    }

    stage('Security Scan (Docker image)') {
      when { expression { return hasDockerfile } }
      steps {
        script {
          echo "Building Docker image ${env.DOCKER_IMAGE}"
          sh "docker build -t ${env.DOCKER_IMAGE} ."
          sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${env.DOCKER_IMAGE} || true"
        }
      }
    }

    stage('Docker Build & Push') {
      when { expression { return params.DOCKER_PUSH && hasDockerfile } }
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'docker-hub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
            sh "docker tag ${env.DOCKER_IMAGE} ${DOCKER_USER}/${env.DOCKER_IMAGE}"
            sh "docker push ${DOCKER_USER}/${env.DOCKER_IMAGE}"
          }
        }
      }
    }

    stage('Notify & Finalize') {
      steps {
        echo "Build finished for ${env.APP_NAME} - build #${env.BUILD_NUMBER}"
        sh 'echo "Branch: ${params.BRANCH}"'
      }
    }

  }

  post {
    success {
      echo "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    unstable {
      echo "UNSTABLE: check test results and warnings"
    }
    failure {
      echo "FAILURE: see logs"
    }
    always {
      cleanWs()
      archiveArtifacts artifacts: '**/target/*.log, **/target/*.xml', allowEmptyArchive: true
    }
  }
}
