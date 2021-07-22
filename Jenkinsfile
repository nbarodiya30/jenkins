def COLOR_MAP = [
    'SUCCESS': '32CD32',
    'FAILURE': 'FF0000',
]

pipeline {
  agent any
  environment {
    APP_NAME = 'rakbank-customer-service'
    DEPLOYMENT_NAME = 'rakdev-customer'
    DOCKER_REPO_NAME = 'customer-service'
    DEPLOYMENT_FILE = 'Dev/frute-deploy-rak-customer-dev.yaml'
    DOCKER_REPO_URL = '772825290081.dkr.ecr.ap-southeast-1.amazonaws.com'
    DOCKER_REPO_CREDENTIALS = 'ecr:ap-southeast-1:jenkins-user'
    DEPLOYMENT_FILES_DIRECTORY = "${WORKSPACE}/deployment_configs/"
    DEPLOYMENT_FILES_GIT_BRANCH = 'jenkins_configurations'
    GIT_BRANCH_NAME = 'rakBank_dev'
    CLUSTER_NAMESPACE = 'Jenkins'
  }

  options {
    // Keep only the last 20 builds
    buildDiscarder(logRotator(numToKeepStr: '20'))

    // Timeout of 1 hour for entire build
    timeout(time: 1, unit: 'HOURS')

    disableConcurrentBuilds()

    
    
  }

  stages {
    stage('Initialization') {
      when { branch env.GIT_BRANCH_NAME }
      steps {
        script {
          // Fetch current tag from ECR and compute new tag name
          env.DOCKER_IMAGE_TAG = sh (
              script: '''
                # Fetching current tag from ECR
                CURRENT_TAG=$(aws ecr describe-images \
                --repository-name ${DOCKER_REPO_NAME} \
                --output text \
                --query 'sort_by(imageDetails,& imagePushedAt)[*].imageTags[*]' \
                | tr '\t' '\n' \
                | tail -1)

                # Fetching correct current tag from ECR if "latest"
                if [ "$CURRENT_TAG" = "latest" ]
                    then
                        CURRENT_TAG=$(aws ecr describe-images \
                        --repository-name ${DOCKER_REPO_NAME} \
                        --output text \
                        --query 'sort_by(imageDetails,& imagePushedAt)[*].imageTags[*]' \
                        | tr '\t' '\n' \
                        | tail -n 2  \
                        | head -n 1)
                fi

                # Fetching correct current tag from ECR if "rakbank_release"
                if echo "$CURRENT_TAG" | grep -q "rakbank_release"
                    then
                        CURRENT_TAG=$(aws ecr describe-images \
                        --repository-name ${DOCKER_REPO_NAME} \
                        --output text \
                        --query 'sort_by(imageDetails,& imagePushedAt)[*].imageTags[*]' \
                        | tr '\t' '\n' \
                        | tail -n 2  \
                        | head -n 1)
                fi

                # Extracting only the last point release number
                ID=${CURRENT_TAG##*.}

                # Incrementing by 1
                NEW_ID=$((ID+1))

                # Computing NEW_TAG by appending NEW_ID to "rakbank-v1.2"
                NEW_TAG=${CURRENT_TAG%.*}.$NEW_ID
                echo $NEW_TAG
              ''',
              returnStdout: true
          ).trim()
          echo "New tag for repo ${DOCKER_REPO_NAME} is: ${DOCKER_IMAGE_TAG}"

          // Fetching author, commit ID and commit message of latest commit
          gitAuthor = sh(
            returnStdout: true,
            script: "git --no-pager show -s --format='%an' ${GIT_COMMIT}"
          ).trim()
          gitCommitId = sh(returnStdout: true, script: "git rev-parse --short ${GIT_COMMIT}").trim()
          gitCommitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
          echo 'Last git commit details:'
          echo 'Author: ' + gitAuthor
          echo 'Commit ID: ' + gitCommitId
          echo 'Commit mesage: ' + gitCommitMessage

          imageWithTag = sh(
            returnStdout: true,
            script: "echo ${DOCKER_REPO_URL}/${DOCKER_REPO_NAME}:${DOCKER_IMAGE_TAG}"
          ).trim()
        }
      }
    }

    stage('Git') {
      when { branch env.GIT_BRANCH_NAME }
      steps {
        echo "App name is ${APP_NAME}"
        git branch: env.GIT_BRANCH_NAME,
          url: "git@bitbucket.org:bambudeveloper/${APP_NAME}.git",
          // GIT_CREDENTIALS is declared as a global environment variable in Jenkins
          credentialsId: env.GIT_CREDENTIALS

        dir(env.DEPLOYMENT_FILES_DIRECTORY) {
          git branch: env.DEPLOYMENT_FILES_GIT_BRANCH,
            url: 'git@bitbucket.org:bambudeveloper/rakbank-frute-backend-setup.git',
            credentialsId: env.GIT_CREDENTIALS
        }
      }
    }

    stage('Docker build') {
      when { branch env.GIT_BRANCH_NAME }
      steps {
        script {
          // --network=host lets Docker know that it can use host network
          sh "docker build --network=host -t ${DOCKER_REPO_NAME} ."
        }
      }
    }
    stage('Docker push') {
      when { branch env.GIT_BRANCH_NAME }
      steps {
        script {
          echo "Tagging image of ${DOCKER_REPO_NAME} as ${env.DOCKER_IMAGE_TAG}"
          echo "Workspace is ${WORKSPACE}"
          // Pushing to AWS ECR Docker registry
          docker.withRegistry('https://${DOCKER_REPO_URL}', env.DOCKER_REPO_CREDENTIALS) {
            // Choosing the image and the registry
            def dockerImage = docker.image(env.DOCKER_REPO_NAME)

            // Tagging with custom tag number (sent as build parameter)
            dockerImage.push(env.DOCKER_IMAGE_TAG)

            // Tagging as 'latest'
            dockerImage.push('latest')
          }
        }
      }
    }
    stage ('Deploy') {
      when { branch env.GIT_BRANCH_NAME }
      steps {
        // Updating KUBECONFIG, setting namespace and applying deployment YAML file
        sh '''
          aws eks --region ap-southeast-1 update-kubeconfig --name frute-backend
          kubectl config set-context --current --namespace=${CLUSTER_NAMESPACE}
          kubectl apply -f ${DEPLOYMENT_FILES_DIRECTORY}${DEPLOYMENT_FILE}

          imageWithTag=${DOCKER_REPO_URL}/${DOCKER_REPO_NAME}:${DOCKER_IMAGE_TAG}
          echo "Image deployed is: " $imageWithTag
          kubectl patch deployment ${DEPLOYMENT_NAME} -p \
          '{"spec":{"template":{"spec":{"containers":[{"image":"'$imageWithTag'","name":"'${DEPLOYMENT_NAME}'"}]}}}}'
        '''
      }
    }
  }
  post {
    cleanup {
      script {
        if (env.DOCKER_IMAGE_TAG) {
          // Deleting locally stored Docker images
          sh '''
            docker rmi ${DOCKER_REPO_URL}/${DOCKER_REPO_NAME}:${DOCKER_IMAGE_TAG}
            docker rmi ${DOCKER_REPO_URL}/${DOCKER_REPO_NAME}:latest
          '''
        }
      }
    }
    always {
      script {
        if (env.DOCKER_IMAGE_TAG) {
          // Sending notification to MS Teams
          office365ConnectorSend (
            // MS_TEAMS_WEBHOOK_URL is declared as a global environment variable in Jenkins
            webhookUrl: env.MS_TEAMS_WEBHOOK_URL,
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}<br /><br />" +
                    "SERVICE: ${APP_NAME}<br /><br />" +
                    "BRANCH: ${GIT_BRANCH_NAME}<br /><br />" +
                    "TAG NAME: ${DOCKER_IMAGE_TAG}<br /><br />" +
                    "NAMESPACE: ${CLUSTER_NAMESPACE}<br /><br />" +
                    "TIME TAKEN: ${currentBuild.durationString.replace(' and counting', '')}<br /><br />" +
                    'LAST COMMIT DETAILS: <br />' +
                    '- Author: ' + gitAuthor + '<br />' +
                    '- Commit ID: ' + gitCommitId + '<br />' +
                    '- Commit mesage: ' + gitCommitMessage + '<br />'
          )
        }
      }
    }
  }
}
