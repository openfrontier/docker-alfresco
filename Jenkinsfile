// 
// https://github.com/jenkinsci/pipeline-model-definition-plugin/wiki/Syntax-Reference
// https://jenkins.io/doc/book/pipeline/syntax/#parallel
// https://jenkins.io/doc/book/pipeline/syntax/#post
pipeline {
    agent any
    environment {
        REPO = 'fjudith/alfresco'
        PRIVATE_REPO = "${PRIVATE_REGISTRY}/${REPO}"
        DOCKER_PRIVATE = credentials('docker-private-registry')
    }
    stages {
        stage ('Checkout') {
            steps {
                script {
                    COMMIT = "${GIT_COMMIT.substring(0,8)}"

                    if ("${BRANCH_NAME}" == "master"){
                        TAG = "latest"
                        ALF_OOO = "libreoffice"
                        ALF_REPO = "repository"
                        ALF_SHA = "share"
                        ALF_SEARCH = "search"
                    }
                    else {
                        TAG = "${BRANCH_NAME}"
                        ALF_OOO = "${BRANCH_NAME}-libreoffice"
                        ALF_REPO = "${BRANCH_NAME}-repository"
                        ALF_SHA = "${BRANCH_NAME}-share"
                        ALF_SEARCH = "${BRANCH_NAME}-search"                        
                    }
                }
                sh 'printenv'
            }
        }
        stage ('Build Alfresco add-ons'){
            parallel {
                stage ("Build Manual Manager add-on"){
                    agent { label 'maven' }
                    steps {
                        git url: 'git://github.com/loftuxab/manual-manager.git',
                            branch: 'master'
                        sh 'tree -sh'
                        sh 'ant package'
                        stash name: 'manual-manager',
                            includes: 'build/dist/**'
                    }
                }
                stage ("Build Markdown Preview add-on"){
                    agent { label 'maven' }
                    steps {
                        // https://bitbucket.org/parashift/alfresco-amp-plugin
                        git url: 'git://github.com/yeyan/alfresco-amp-plugin.git',
                            branch: 'master'
                        sh 'tree -sh'
                        sh 'gradle publish'
                        git url: 'git://github.com/fjudith/md-preview.git',
                            branch: '1.7.2'
                        sh 'tree -sh'
                        sh 'cd share/ && gradle amp && cd ../'
                        sh 'cd repo/ && gradle amp && cd ../'
                        stash name: 'md-preview',
                            includes: 'repo/build/amp/**,share/build/amp/**'
                    }
                }
            }
            stage ('Alfresco Web & Application server') {
                agent { label 'docker'}
                steps {
                    sh "docker build -f slim/Dockerfile -t ${REPO}:${COMMIT} slim/"
                    sh "docker run -d --name 'alfresco-${BUILD_NUMBER}' -p 55080:8080 -p 55443:8443 ${REPO}:${COMMIT}"
                    sh "docker ps -a"
                    sleep 300
                    sh "docker logs alfresco-${BUILD_NUMBER}"
                    sh "docker run --rm --link alfresco-${BUILD_NUMBER}:alfresco blitznote/debootstrap-amd64:17.04 bash -c \"curl -i -X GET -u admin:admin http://alfresco:8080/alfresco/service/api/audit/control\""
                }
                post {
                    success {
                        echo 'Tag and Push to private registry'
                        sh "docker tag ${REPO}:${COMMIT} ${PRIVATE_REPO}:${TAG}"
                    }
                }
            }
        }
    }
    stages {
        stage ('Alfresco LibreOffice') {
            parallel {
                stage ('Docker build Micro-Service'){
                
                    agent { label 'docker'}
                    steps {
                        sh "docker build -f libreoffice/Dockerfile -t ${REPO}:${COMMIT}-libreoffice libreoffice/"
                    }
                    post {
                        success {
                            echo 'Tag for private registry'
                            sh "docker tag ${REPO}:${COMMIT}-libreoffice ${PRIVATE_REPO}:${ALF_OOO}"
                        }
                    }
                }
                stage ('Alfresco Search Services') {
                    agent { label 'docker'}
                    steps {
                        sh "docker build -f search/Dockerfile -t ${REPO}:${COMMIT}-search search/"
                    }
                    post {
                        success {
                            echo 'Tag for private registry'
                            sh "docker tag ${REPO}:${COMMIT}-search ${PRIVATE_REPO}:${ALF_SEARCH}"
                        }
                    }
                }
                stage ('Alfresco Content Repository Services') {
                    agent { label 'docker'}
                    steps {
                        sh "docker build -f repository/Dockerfile -t ${REPO}:${COMMIT}-repository repository/"
                    }
                    post {
                        success {
                            echo 'Tag for private registry'
                            sh "docker tag ${REPO}:${COMMIT}-repository ${PRIVATE_REPO}:${ALF_REPO}"
                        }
                    }
                }
                stage ('Alfresco Share') {
                    agent { label 'docker'}
                    steps {
                        sh "docker build -f share/Dockerfile -t ${REPO}:${COMMIT}-share share/"
                    }
                    post {
                        success {
                            echo 'Tag for private registry'
                            sh "docker tag ${REPO}:${COMMIT}-share ${PRIVATE_REPO}:${ALF_SHA}"
                        }
                    }
                }
            }
        }
    }
    stages {
        stage ('RUN'){
            parallel {
                stage ('Slim'){
                    agent { label 'docker' }
                    steps {
                        // Start database
                        sh "docker run -d --name 'mysql-${BUILD_NUMBER}' -e MYSQL_USER=alfresco -e MYSQL_PASSWORD=alfresco -e MYSQL_DATABASE=alfresco amd64/mysql:5.6"
                        // Start application
                        sh "docker run -d --name 'alfresco-${BUILD_NUMBER}' --link mysql-${BUILD_NUMBER}:mysql ${REPO}:${COMMIT}"
                    }
                }
                stage ('Micro-Services'){
                    agent { label 'docker'}
                    steps {
                        // Start database
                        sh "docker run -d --name 'postgres-${BUILD_NUMBER}' -e POSTGRES_USER=alfresco -e POSTGRES_PASSWORD=alfresco -e POSTGRES_DB=alfresco amd64/postgres:9.4"
                        //Start application micro-services
                        sh "docker run -d --name 'libreoffice-${BUILD_NUMBER}' -p 56082:8100 ${REPO}:${COMMIT}-libreoffice"
                        sh "docker run -d --name 'search-${BUILD_NUMBER}' -p 56082:8100 ${REPO}:${COMMIT}-search"
                        sh "docker run -d --name 'repository-${BUILD_NUMBER}' --link postgres-${BUILD_NUMBER}:postgres --link libreoffice-${BUILD_NUMBER}:libreoffice --link search-${BUILD_NUMBER}:search -p 56080:8080 -p 56443:8443 ${REPO}:${COMMIT}-repository"
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Run regardless of the completion status of the Pipeline run.'
            echo 'Remove slim stack'
            sh "docker rm -f alfresco-${BUILD_NUMBER}"
            sh "docker rm -f mysql-${BUILD_NUMBER}"
            echo 'Remove micro-services stack'
            sh "docker rm -f postgres-${BUILD_NUMBER}"
            sh "docker rm -f share-${BUILD_NUMBER}"
            sh "docker rm -f repository-${BUILD_NUMBER}"
            sh "docker rm -f search-${BUILD_NUMBER}"
            sh "docker rm -f libreoffice-${BUILD_NUMBER}"
        }
        changed {
            echo 'Only run if the current Pipeline run has a different status from the previously completed Pipeline.'
        }
        success {
            echo 'Only run if the current Pipeline has a "success" status, typically denoted in the web UI with a blue or green indication.'
            sh "docker login -u ${DOCKER_PRIVATE_USR} -p ${DOCKER_PRIVATE_PSW} ${PRIVATE_REGISTRY}"
            sh "docker push ${PRIVATE_REPO}"
        }
        unstable {
            echo 'Only run if the current Pipeline has an "unstable" status, usually caused by test failures, code violations, etc. Typically denoted in the web UI with a yellow indication.'
        }
        aborted {
            echo 'Only run if the current Pipeline has an "aborted" status, usually due to the Pipeline being manually aborted. Typically denoted in the web UI with a gray indication.'
        }
    }
}