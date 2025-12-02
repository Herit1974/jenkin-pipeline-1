pipeline {
  // let Jenkins pick an available agent (node). Docker must be available on node for some steps.
  agent any

  // Make pipeline behavior nicer & reproducible
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(true) // we'll checkout explicitly
  }

  // parameters let you change behavior from Jenkins UI when you run the job
  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    booleanParam(name: 'RUN_INTEGRATION_TESTS', defaultValue: true, description: 'Run integration tests?')
    booleanParam(name: 'DOCKER_PUSH', defaultValue: false, description: 'Push Docker image to registry after build?')
  }

  environment {
    // example env vars (use Jenkins Credentials for secrets)
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
          userRemoteConfigs: [[url: 'https://github.com/yourname/your-repo.git']]
        ])
      }
    }

    stage('Prepare') {
      steps {
        script {
          // detect project type quickly
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
            sh 'mvn -B -DskipTests clean verify' // runs checks/plugins configured in pom
            // You can add spotbugs/checkstyle in pom for more checks
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
            sh 'echo "No recognised build (pom.xml or package.json missing). Creating marker file."'
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
            sh 'npm test || true' // keep pipeline running (or remove || true to fail)
            junit 'test-results/*.xml' // adjust to your test reporter
          } else {
            echo 'No unit tests to run'
          }
        }
      }
      post {
        always {
          // keep a short test summary
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
                // run maven integration profile that maybe starts containers via surefire/failsafe plugins
                sh 'mvn -B -Pintegration verify || true'
                junit '**/target/failsafe-reports/*.xml'
              } else {
                echo 'No integration tests for non-Maven project configured'
              }
            }
          }
        }

        stage('Integration: API Endpoints') {
          steps {
            script {
              // example: run a small docker compose to provide dependencies or run tests against an endpoint
              sh 'echo "Pretend: start test services and run API tests"'
              // you could use 'docker-compose -f docker-compose.test.yml up --exit-code-from test'
            }
          }
        }
      }
    }

    stage('Static Analysis / Sonar (optional)') {
      when { expression { return env.SONAR_HOST_URL != null } } // set SONAR_HOST_URL in Jenkins if you have it
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
          // Build local image (needs docker on agent)
          echo "Building Docker image ${env.DOCKER_IMAGE}"
          sh "docker build -t ${env.DOCKER_IMAGE} ."

          // Run a Trivy scan if trivy is available (or container image). This is an example.
          // If Trivy isn't installed on the agent, you can run via docker:
          sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${env.DOCKER_IMAGE} || true"
          // Note: scanner exit codes may fail the build depending on config; here we ignore errors (|| true) to continue pipeline.
        }
      }
    }

    stage('Docker Build & Push') {
      when { expression { return params.DOCKER_PUSH && hasDockerfile } }
      steps {
        script {
          // Push to DockerHub (example). You must add a Jenkins credential of type "Username with password" with id 'docker-hub-cred'
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
        sh 'echo "Build details:" && echo "Branch: ${params.BRANCH}" && echo "Build: ${env.BUILD_NUMBER}"'
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
      echo "FAILURE: see console output for errors"
    }
    always {
      cleanWs()
      archiveArtifacts artifacts: '**/target/*.log, **/target/*.xml', allowEmptyArchive: true
    }
  }
}
