  @Library('common-shared') _

  pipeline {
    agent any

    environment {
      APP_NAME = 'git-eca-rest-api'
      NAMESPACE = 'foundation-internal-webdev-apps'
      IMAGE_NAME = 'eclipsefdn/git-eca-rest-api'
      CONTAINER_NAME = 'app'
      ENVIRONMENT = sh(
        script: """
          if [ "${env.BRANCH_NAME}" = "master" ]; then
            printf "production"
          else
            printf "${env.BRANCH_NAME}"
          fi
        """,
        returnStdout: true
      )
      TAG_NAME = sh(
        script: """
          GIT_COMMIT_SHORT=\$(git rev-parse --short ${env.GIT_COMMIT})
          if [ "${env.ENVIRONMENT}" = "" ]; then
            printf \${GIT_COMMIT_SHORT}-${env.BUILD_NUMBER}
          else
            printf ${env.ENVIRONMENT}-\${GIT_COMMIT_SHORT}-${env.BUILD_NUMBER}
          fi
        """,
        returnStdout: true
      )
    }

    options {
      buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
      stage('Build Java code') {
        steps {
          sh './mvnw -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn --batch-mode package'
          stash includes: 'target/', name: 'target'
        }
      }

      stage('Build docker image') {
        agent {
          label 'docker-build'
        }
        steps {
          unstash 'target'

          sh 'docker build -f src/main/docker/Dockerfile.jvm --no-cache -t ${IMAGE_NAME}:${TAG_NAME} -t ${IMAGE_NAME}:latest .'
        }
      }

      stage('Push docker image') {
        agent {
          label 'docker-build'
        }
        when {
          anyOf {
            environment name: 'ENVIRONMENT', value: 'production'
            environment name: 'ENVIRONMENT', value: 'staging'
          }
        }
        steps {
          withDockerRegistry([credentialsId: '04264967-fea0-40c2-bf60-09af5aeba60f', url: 'https://index.docker.io/v1/']) {
            sh '''
              docker push ${IMAGE_NAME}:${TAG_NAME}
              docker push ${IMAGE_NAME}:latest
            '''
          }
        }
      }

      stage('Deploy to cluster') {
        agent {
          kubernetes {
            label 'kubedeploy-agent'
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: kubectl
                image: eclipsefdn/kubectl:1.9-alpine
                command:
                - cat
                tty: true
            '''
          }
        }

        when {
          anyOf {
            environment name: 'ENVIRONMENT', value: 'production'
            environment name: 'ENVIRONMENT', value: 'staging'
          }
        }
        steps {
          container('kubectl') {
            withKubeConfig([credentialsId: '1d8095ea-7e9d-4e94-b799-6dadddfdd18a', serverUrl: 'https://console-int.c1-ci.eclipse.org']) {
              sh '''
                DEPLOYMENT="$(k8s getFirst deployment "${NAMESPACE}" "app=${APP_NAME},environment=${ENVIRONMENT}")"
                if [[ $(echo "${DEPLOYMENT}" | jq -r 'length') -eq 0 ]]; then
                  echo "ERROR: Unable to find a deployment to patch matching 'app=${APP_NAME},environment=${ENVIRONMENT}' in namespace ${NAMESPACE}"
                  exit 1
                else 
                  DEPLOYMENT_NAME="$(echo "${DEPLOYMENT}" | jq -r '.metadata.name')"
                  kubectl set image "deployment.v1.apps/${DEPLOYMENT_NAME}" -n "${NAMESPACE}" "${CONTAINER_NAME}=${IMAGE_NAME}:${TAG_NAME}" --record=true
                  if ! kubectl rollout status "deployment.v1.apps/${DEPLOYMENT_NAME}" -n "${NAMESPACE}"; then
                    # will fail if rollout does not succeed in less than .spec.progressDeadlineSeconds
                    kubectl rollout undo "deployment.v1.apps/${DEPLOYMENT_NAME}" -n "${NAMESPACE}"
                    exit 1
                  fi
                fi
              '''
            }
          }
        }
      }
    }

    post {
      always {
        deleteDir() /* clean up workspace */
        sendNotifications currentBuild
      }
    }
  }
