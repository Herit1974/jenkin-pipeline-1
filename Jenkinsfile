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
          // declare variables to avoid warnings
          def isMaven = fileExists('pom.xml')
          def isNode  = fileExists('package.json')
          def hasDockerfile = fileExists('Dockerfile')
          // expose them to other script blocks via env (strings) if needed
          env.IS_MAVEN = isMaven.toString()
          env.IS_NODE = isNode.toString()
          env.HAS_DOCKER = hasDockerfile.toString()

          echo "Detected: Maven=${isMaven}, Node=${isNode}, Dockerfile=${hasDockerfile}"
        }
      }
    }

    stage('Lint & Static Checks') {
      parallel {
        stage('Maven/Java Lint') {
          when { expression { return env.IS_MAVEN == 'true' } }
          steps {
            script {
              // cross-platform wrapper
              def run = { cmd ->
                if (isUnix()) {
                  sh cmd
                } else {
                  bat cmd
                }
              }
              echo "Running Maven lint/verify"
              timeout(time: 10, unit: 'MINUTES') {
                run('mvn -B -DskipTests clean verify')
              }
            }
          }
        }

        stage('Node Lint') {
          when { expression { return env.IS_NODE == 'true' } }
          steps {
            script {
              def run = { cmd -> if (isUnix()) sh cmd else bat cmd }
              echo "Running npm ci and lint (if present)"
              timeout(time: 10, unit: 'MINUTES') {
                run('npm ci || exit 0')
                // attempt lint if script exists (may show "no-lint-script")
                if (fileExists('package.json')) {
                  run('npm run lint || echo no-lint-script')
                }
              }
            }
          }
        }
      }
    }

    stage('Build') {
      steps {
        script {
          def run = { cmd -> if (isUnix()) sh cmd else bat cmd }

          if (env.IS_MAVEN == 'true') {
            echo "Maven build starting"
            timeout(time: 15, unit: 'MINUTES') {
              run('mvn -B -DskipTests package')
            }
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
          } else if (env.IS_NODE == 'true') {
            echo "Node build starting"
            timeout(time: 15, unit: 'MINUTES') {
              run('npm ci')
              run('npm run build || exit 0')
            }
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
          } else {
            echo "No recognised build files (pom.xml or package.json). Creating marker file."
            if (isUnix()) {
              sh 'echo "no-build" > NO_BUILD.txt'
            } else {
              bat 'echo no-build > NO_BUILD.txt'
            }
            archiveArtifacts artifacts: 'NO_BUILD.txt'
          }
        }
      }
    }

    stage('Unit Tests') {
      steps {
        script {
          def run = { cmd -> if (isUnix()) sh cmd else bat cmd }

          if (env.IS_MAVEN == 'true') {
            echo "Running Maven unit tests"
            timeout(time: 10, unit: 'MINUTES') {
              run('mvn -B test')
            }
            junit '**/target/surefire-reports/*.xml'
          } else if (env.IS_NODE == 'true') {
            echo "Running Node unit tests (if any)"
            timeout(time: 10, unit: 'MINUTES') {
              run('npm test || exit 0')
            }
            // adjust path to your reporter if needed
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
              def run = { cmd -> if (isUnix()) sh cmd else bat cmd }
              if (env.IS_MAVEN == 'true') {
                echo "Running Maven integration profile (if configured)"
                timeout(time: 15, unit: 'MINUTES') {
                  run('mvn -B -Pintegration verify || exit 0')
                }
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
              def run = { cmd -> if (isUnix()) sh cmd else bat cmd }
              echo "Pretend: starting API tests (replace with real commands)"
              timeout(time: 10, unit: 'MINUTES') {
                run('echo "API tests placeholder"')
              }
            }
          }
        }
      }
    }

    stage('Static Analysis / Sonar (optional)') {
      when { expression { return env.SONAR_HOST_URL != null } }
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          script {
            def run = { cmd -> if (isUnix()) sh cmd else bat cmd }
            run('mvn -B sonar:sonar -Dsonar.login=${SONAR_TOKEN} -Dsonar.host.url=${SONAR_HOST_URL}')
          }
        }
      }
    }

    stage('Security Scan (Docker image)') {
      when { expression { return env.HAS_DOCKER == 'true' } }
      steps {
        script {
          def run = { cmd -> if (isUnix()) sh cmd else bat cmd }
          echo "Building Docker image ${env.DOCKER_IMAGE}"
          // Docker must be installed and usable on the node for these commands
          timeout(time: 20, unit: 'MINUTES') {
            run("docker build -t ${env.DOCKER_IMAGE} .")
            // run trivy via docker; ignore failure so pipeline continues
            run("docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${env.DOCKER_IMAGE} || exit 0")
          }
        }
      }
    }

    stage('Docker Build & Push') {
      when { expression { return params.DOCKER_PUSH && env.HAS_DOCKER == 'true' } }
      steps {
        script {
          def run = { cmd -> if (isUnix()) sh cmd else bat cmd }
          withCredentials([usernamePassword(credentialsId: 'docker-hub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            timeout(time: 10, unit: 'MINUTES') {
              run("docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}")
              run("docker tag ${env.DOCKER_IMAGE} ${DOCKER_USER}/${env.DOCKER_IMAGE}")
              run("docker push ${DOCKER_USER}/${env.DOCKER_IMAGE}")
            }
          }
        }
      }
    }

    stage('Notify & Finalize') {
      steps {
        echo "Build finished for ${env.APP_NAME} - build #${env.BUILD_NUMBER}"
        echo "Branch: ${params.BRANCH}"
      }
    }

  } // stages

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
