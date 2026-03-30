pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    ansiColor('xterm')
  }

  tools {
    maven 'maven3'              // Configure in Jenkins Global Tool Configuration
    // jdk 'jdk17'              // Optional if you manage JDK via Jenkins tools
  }

  environment {
    // ---- App ----
    APP_NAME        = "myapp"
    IMAGE_TAG       = "${BUILD_NUMBER}"          // or use Git SHA short
    DOCKERFILE_PATH = "Dockerfile"

    // ---- Sonar ----
    SONARQUBE_SERVER = "My SonarQube Server"     // Jenkins: Manage Jenkins -> Configure System -> SonarQube servers

    // ---- Nexus ----
    NEXUS_DOCKER_REGISTRY = "nexus.company.com:8082"   // Docker hosted repo endpoint
    NEXUS_DOCKER_REPO     = "docker-hosted"            // repo name/path if applicable
    NEXUS_IMAGE           = "${NEXUS_DOCKER_REGISTRY}/${NEXUS_DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}"

    // ---- JFrog / Artifactory ----
    JFROG_DOCKER_REGISTRY = "company.jfrog.io"
    JFROG_DOCKER_REPO     = "docker-local"
    JFROG_IMAGE           = "${JFROG_DOCKER_REGISTRY}/${JFROG_DOCKER_REPO}/${APP_NAME}:${IMAGE_TAG}"

    // ---- GitOps (Argo CD source) ----
    GITOPS_REPO_URL   = "https://github.com/myorg/app-gitops.git"
    GITOPS_BRANCH     = "main"
    GITOPS_APP_PATH   = "apps/myapp"

    // ---- Argo CD ----
    ARGOCD_SERVER     = "argocd.company.com"
    ARGOCD_APP_NEXUS  = "myapp-nexus"
    ARGOCD_APP_JFROG  = "myapp-jfrog"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Maven Build') {
      steps {
        sh "mvn -U -B clean compile"
      }
    }

    stage('Unit Tests') {
      steps {
        sh "mvn -B test"
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: "**/target/surefire-reports/*.xml"
        }
      }
    }

    stage('Functional / Integration Tests') {
      steps {
        // Use Maven Failsafe with a profile (recommended) or your own command
        sh "mvn -B verify -Pintegration-tests"
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: "**/target/failsafe-reports/*.xml"
        }
      }
    }

    stage('Performance Tests') {
      steps {
        // Example placeholders:
        // - JMeter: mvn -Pperf jmeter:jmeter
        // - Gatling: mvn -Pperf gatling:test
        sh "mvn -B verify -Pperformance-tests"
      }
    }

    stage('Sonar Scan + Quality Gate') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          // SonarQube Jenkins steps are documented: withSonarQubeEnv + waitForQualityGate [4](https://www.jenkins.io/doc/pipeline/steps/sonar/)
          withSonarQubeEnv("${SONARQUBE_SERVER}") {
            sh """
              mvn -B sonar:sonar \
                -Dsonar.login=$SONAR_TOKEN \
                -Dsonar.projectKey=${APP_NAME} \
                -Dsonar.projectName=${APP_NAME}
            """
          }
        }
      }
    }

    stage('Quality Gate (Sonar)') {
      steps {
        timeout(time: 30, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh """
          docker build -t ${NEXUS_IMAGE} -f ${DOCKERFILE_PATH} .
          docker tag ${NEXUS_IMAGE} ${JFROG_IMAGE}
        """
      }
    }

    stage('Push Images (Nexus & JFrog)') {
      parallel {

        stage('Push to Nexus Docker Registry') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
              // docker build+push to Nexus registry is commonly done via docker login/push [7](https://bhairavisanskriti.hashnode.dev/publish-docker-images-to-nexus-using-jenkins)[8](https://stackoverflow.com/questions/58200072/how-to-push-docker-image-to-nexus-repository-through-jenkinsfile)
              sh """
                echo "$NEXUS_PASS" | docker login ${NEXUS_DOCKER_REGISTRY} -u "$NEXUS_USER" --password-stdin
                docker push ${NEXUS_IMAGE}
                docker logout ${NEXUS_DOCKER_REGISTRY}
              """
            }
          }
        }

        stage('Push to JFrog Artifactory Docker Registry') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'jfrog-docker-creds', usernameVariable: 'JFROG_USER', passwordVariable: 'JFROG_PASS')]) {
              // Example approach: docker.withRegistry is widely used for JFrog docker pushes [9](https://devopspilot.com/jenkins/pipelines/docker-plugin-build-deploy-jfrog/)[5](https://plugins.jenkins.io/jfrog/)
              sh """
                echo "$JFROG_PASS" | docker login ${JFROG_DOCKER_REGISTRY} -u "$JFROG_USER" --password-stdin
                docker push ${JFROG_IMAGE}
                docker logout ${JFROG_DOCKER_REGISTRY}
              """
            }
          }
        }

      }
    }

    stage('Update GitOps manifests (for Argo CD)') {
      steps {
        withCredentials([string(credentialsId: 'gitops-pat', variable: 'GITOPS_PAT')]) {
          sh """
            rm -rf gitops && git clone -b ${GITOPS_BRANCH} https://${GITOPS_PAT}@${GITOPS_REPO_URL.replace('https://','')} gitops
            cd gitops/${GITOPS_APP_PATH}

            # Update Nexus overlay tag
            # (Assumes overlays/nexus/kustomization.yaml uses kustomize 'images:' entries)
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newTag = "${IMAGE_TAG}")' overlays/nexus/kustomization.yaml
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newName = "${NEXUS_DOCKER_REGISTRY}/${NEXUS_DOCKER_REPO}/${APP_NAME}")' overlays/nexus/kustomization.yaml

            # Update JFrog overlay tag
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newTag = "${IMAGE_TAG}")' overlays/jfrog/kustomization.yaml
            yq -i '.images[] |= (select(.name == "${APP_NAME}").newName = "${JFROG_DOCKER_REGISTRY}/${JFROG_DOCKER_REPO}/${APP_NAME}")' overlays/jfrog/kustomization.yaml

            git config user.email "ci-bot@company.com"
            git config user.name "ci-bot"
            git add .
            git commit -m "chore(${APP_NAME}): bump image tag to ${IMAGE_TAG}" || true
            git push origin ${GITOPS_BRANCH}
          """
        }
      }
    }

    stage('Argo CD Sync (Nexus & JFrog apps)') {
      steps {
        withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {

          // In GitOps, ArgoCD syncs after Git changes. Triggering sync is optional but common. [1](https://oneuptime.com/blog/post/2026-02-26-argocd-image-tag-update-workflow/view)[2](https://argocd-image-updater.readthedocs.io/en/stable/)
          sh """
            argocd login ${ARGOCD_SERVER} --auth-token $ARGOCD_TOKEN --insecure

            argocd app sync ${ARGOCD_APP_NEXUS}
            argocd app wait ${ARGOCD_APP_NEXUS} --health --timeout 300

            argocd app sync ${ARGOCD_APP_JFROG}
            argocd app wait ${ARGOCD_APP_JFROG} --health --timeout 300
          """
        }
      }
    }
  }

  post {
    always {
      sh "docker image prune -f || true"
    }
    success {
      echo "✅ Pipeline completed: build+tests+sonar+docker push+argo deploy"
    }
    failure {
      echo "❌ Pipeline failed. Check stage logs."
    }
  }
}