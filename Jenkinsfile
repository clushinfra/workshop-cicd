pipeline {
    agent any

    environment {
        // 기본 설정
        IMAGE_REPO_URL    = "27.96.145.28:30500"
        ARGOCD_URL        = "27.96.145.28:30938"
        DEPLOY_PATH       = "${DEPLOY_APP_NAME}/cicd/k8s"

        // 파생 변수
        IMAGE_NAME        = "${IMAGE_REPO_URL}/${NS}-${CICD_BRANCH}-${DEPLOY_APP_NAME}"
        ARGOCD_APP_NAME   = "${NS}-${CICD_BRANCH}-${DEPLOY_APP_NAME}"
    }

    stages {
        stage('Git Clone') {
            steps {
                // 소스 코드 저장소에서 Git clone
                withCredentials([gitUsernamePassword(credentialsId: 'GITHUB', gitToolName: 'Default')]) {
                    cleanWs()
                    git credentialsId: 'GITHUB', url: "https://${SOURCE_GIT_URL}", branch: "${SOURCE_BRANCH}"
                    
                    script {
                        def commitHash = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
                        IMAGE_TAG = commitHash
                        IMAGE_NAME = "${IMAGE_NAME}:${IMAGE_TAG}-${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Image Build & Push') {
            steps {
                // Docker 이미지 빌드 및 Nexus에 Push
                withCredentials([usernamePassword(credentialsId: 'NEXUS', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PSW')]) {
                    sh "buildah login --tls-verify=false -u ${NEXUS_USER} -p ${NEXUS_PSW} ${IMAGE_REPO_URL}"

                    sh """
                    cat <<EOF > Dockerfile
FROM ${DOCKER_BASE_IMAGE}
RUN mkdir -p /usr/local/nginx/html
COPY workshop.png /usr/local/nginx/html/
COPY workshop.html /usr/local/nginx/html/index.html
RUN chmod -R 755 /usr/local/nginx/html
EXPOSE ${DOCKER_DEPLOY_PORT}
EOF
                    """

                    sh """
                    buildah bud --isolation chroot -t ${IMAGE_NAME} -f Dockerfile .
                    buildah push --tls-verify=false ${IMAGE_NAME}
                    buildah rmi ${IMAGE_NAME}
                    """
                }
            }
        }

        stage('Image Tag Update') {
            steps {
                // deploy.yaml 내 이미지 태그 변경 후 Git push
                withCredentials([gitUsernamePassword(credentialsId: 'GITHUB', gitToolName: 'Default')]) {
                    git credentialsId: 'GITHUB', url: "https://${CICD_GIT_URL}", branch: "${CICD_BRANCH}"

                    sh """
                    git config --global user.name clushinfra
                    git config --global user.email infra@clush.net

                    sed -i -E 's%image:.+%image: ${IMAGE_NAME}%' ${DEPLOY_PATH}/deploy.yaml

                    git add ${DEPLOY_PATH}/deploy.yaml
                    git commit --allow-empty -m '[UPDATE] ${DEPLOY_APP_NAME} ${IMAGE_TAG} Deploy'
                    git push origin ${CICD_BRANCH}
                    """
                }
            }
        }

        stage('ArgoCD Sync') {
            steps {
                // ArgoCD를 통해 클러스터에 자동 배포
                withCredentials([
                    usernamePassword(credentialsId: 'GITHUB', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN'),
                    usernamePassword(credentialsId: 'ARGOCD', usernameVariable: 'ARGOCD_USER', passwordVariable: 'ARGOCD_PSW')
                ]) {
                    sh """
                    argocd login ${ARGOCD_URL} --username ${ARGOCD_USER} --password ${ARGOCD_PSW} --grpc-web --insecure

                    argocd repo add https://${CICD_GIT_URL} --username ${GIT_USER} --password ${GIT_TOKEN} --grpc-web

                    argocd proj create ${NS} --src='*' --dest='*',${NS} --upsert --grpc-web
                    argocd proj allow-cluster-resource ${NS} '*' '*' --grpc-web

                    argocd app create ${ARGOCD_APP_NAME} --project ${NS} \\
                      --repo https://${CICD_GIT_URL} --revision ${CICD_BRANCH} --path ${DEPLOY_PATH} \\
                      --dest-server ${CLUSTER} --dest-namespace ${NS} \\
                      --sync-option CreateNamespace=true --grpc-web

                    argocd app sync ${ARGOCD_APP_NAME} --project ${NS} --grpc-web --prune
                    """
                }
            }
        }
    }
}
