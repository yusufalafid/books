pipeline {
    options {
        // set a timeout of 60 minutes for this pipeline
        timeout(time: 60, unit: 'MINUTES')
    }
    agent {
      node {
        label 'nodejs'
      }
    }

    environment {
        //TODO: Edit these vars as per your env
        DEV_PROJECT = "ucup-dev"
        STAGE_PROJECT = "ucup-stage"
        APP_GIT_URL = "https://github.com/yusufalafid/books"
        NEXUS_SERVER = "http://192.168.10.26:8081/repository/npm-group/"

        // DO NOT CHANGE THE GLOBAL VARS BELOW THIS LINE
        APP_NAME = "books"
    }


    stages {

        stage('NPM Install') {
            steps {
                echo '### Installing NPM dependencies ###'
                sh '''
                        npm config set registry ${NEXUS_SERVER}
                        npm install
                   '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo '### Running unit tests ###'
                sh 'npm test'
            }
        }

        stage('Launch new app in DEV env') {
            steps {
                echo '### Cleaning existing resources in DEV env ###'
                sh '''
                        oc project ${DEV_PROJECT}
                        oc delete all -l app=${APP_NAME}
                        sleep 5
                   '''

                echo '### Creating a new app in DEV env ###'
                sh '''
                        oc project ${DEV_PROJECT}
                        oc new-app --name books dominicgebhart/s2i-nodejs8-centos7-xvfb~${APP_GIT_URL} \
                        --build-env npm_config_registry=${NEXUS_SERVER} \
                        --allow-missing-images

                        oc expose svc/${APP_NAME}
                   '''
            }
        }

        stage('Wait for S2I build to complete') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject( "${DEV_PROJECT}" ) {
                            def bc = openshift.selector("bc", "${APP_NAME}")
                            bc.logs('-f')
                            def builds = bc.related('builds')
                            builds.untilEach(1) {
                                return (it.object().status.phase == "Complete")
                            }
                        }
                    }
                }
            }
        }

        stage('Wait for deployment in DEV env') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject( "${DEV_PROJECT}" ) {
                            def deployment = openshift.selector("dc", "${APP_NAME}").rollout()
                            openshift.selector("dc", "${APP_NAME}").related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                }
            }
        }

        stage('Promote to Staging Env') {
            steps {
                timeout(time: 60, unit: 'MINUTES') {
                    input message: "Promote to Staging?"
                }
                script {
                    openshift.withCluster(){
                        openshift.tag("${DEV_PROJECT}/books:latest", "${STAGE_PROJECT}/books:stage")
                    }
                }
            }
        }

        stage('Deploy to Staging Env') {
            steps {
                echo '### Cleaning existing resources in Staging ###'
                sh '''
                        oc project ${STAGE_PROJECT}
                        oc delete all -l app=${APP_NAME}
                        sleep 5
                   '''

                echo '### Creating a new app in Staging ###'
                sh '''
                        oc project ${STAGE_PROJECT}
                        oc new-app --name books -i books:stage

                        oc expose svc/${APP_NAME}
                   '''
            }
        }

        stage('Wait for deployment in Staging') {
            steps {
                sh "oc get route ${APP_NAME} -n ${STAGE_PROJECT} -o jsonpath='{ .spec.host }' --loglevel=4 > routehost"

                script {
                    routeHost = readFile('routehost').trim()

                    openshift.withCluster() {
                        openshift.withProject( "${STAGE_PROJECT}" ) {
                            def deployment = openshift.selector("dc", "${APP_NAME}").rollout()
                            openshift.selector("dc", "${APP_NAME}").related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }

                        echo "Deployment to Staging env is complete. Access the app at the URL http://${routeHost}."
                    }
                }
            }
        }
    }
}
