pipeline {
  agent any

  environment {
    IMAGE_NAME = "eeacms/plone-backend"
    GIT_NAME   = "plone-backend"
  }

  parameters {
    string(name: 'TARGET_BRANCH', defaultValue: '', description: 'Run tests with GIT_BRANCH env enabled')
  }

  stages {
    stage('Build & Test') {
      environment {
        // sanitize BUILD_TAG: lowercase + replace spaces with dashes
        TAG = "${BUILD_TAG.toLowerCase().replaceAll('\\s+', '-')}"
      }
      steps {
        script {
          try {
            checkout scm
            sh 'docker -v'
            sh 'hostname'
            sh 'ls -l && pwd'

            // Build Docker image, quoting the image:tag as a single argument
            sh "docker build --no-cache -t \"${IMAGE_NAME}:${TAG}\" ."

            // Run tests
            sh "./test/run.sh \"${IMAGE_NAME}:${TAG}\""
          } finally {
            // Cleanup the built image (ignore failure if not found)
            sh script: "docker rmi \"${IMAGE_NAME}:${TAG}\"", returnStatus: true
          }
        }
      }
    }

    stage('Release on tag creation') {
      when { buildingTag() }
      steps {
        withCredentials([
          string(credentialsId: 'eea-jenkins-token',   variable: 'GITHUB_TOKEN'),
          string(credentialsId: 'plone-backend-trigger', variable: 'TRIGGER_MAIN_URL'),
          usernamePassword(credentialsId: 'jekinsdockerhub', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')
        ]) {
          sh """
            docker pull eeacms/gitflow
            docker run -i --rm --name="${BUILD_TAG}" \
              -e GIT_BRANCH="${BRANCH_NAME}" \
              -e GIT_NAME="${GIT_NAME}" \
              -e DOCKERHUB_REPO="eeacms/plone-backend" \
              -e GIT_TOKEN="${GITHUB_TOKEN}" \
              -e DOCKERHUB_USER="${DOCKERHUB_USER}" \
              -e DOCKERHUB_PASS="${DOCKERHUB_PASS}" \
              -e TRIGGER_MAIN_URL="${TRIGGER_MAIN_URL}" \
              -e DEPENDENT_DOCKERFILE_URL="eea/eea-website-backend/blob/master/Dockerfile eea/fise-backend/blob/master/Dockerfile eea/advisory-board-backend/blob/master/Dockerfile eea/bise-backend/blob/plone-6/Dockerfile" \
              -e GITFLOW_BEHAVIOR="RUN_ON_TAG" \
              eeacms/gitflow
          """
        }
      }
    }
  }

  post {
    always {
      cleanWs(
        cleanWhenAborted: true,
        cleanWhenFailure: true,
        cleanWhenNotBuilt: true,
        cleanWhenSuccess: true,
        cleanWhenUnstable: true,
        deleteDirs: true
      )
    }
    changed {
      script {
        def details = """
          <h1>${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}</h1>
          <p>Check console output at <a href="${env.BUILD_URL}/display/redirect">${env.JOB_BASE_NAME} - #${env.BUILD_NUMBER}</a></p>
        """
        emailext(
          subject: '$DEFAULT_SUBJECT',
          body: details,
          attachLog: true,
          compressLog: true,
          recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'CulpritsRecipientProvider']]
        )
      }
    }
  }
}
