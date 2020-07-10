pipeline{

    agent{
        // label "" also could have been 'agent any' - that has the same meaning.
        label "master"
    }

    environment {
        // Global Vars
        NAMESPACE_PREFIX="bnb"
        GIT_DOMAIN = ""
        GIT_ORG = ""

        PIPELINES_NAMESPACE = "${NAMESPACE_PREFIX}-ci-cd"
        APP_NAME = "application"

        JENKINS_TAG = "${JOB_NAME}.${BUILD_NUMBER}".replace("/", "-")
        JOB_NAME = "${JOB_NAME}".replace("/", "-")

        GIT_SSL_NO_VERIFY = true
        GIT_CREDENTIALS = credentials("${GIT_ORG}-ci-cd-gitlab-auth")

        DISPLAY = 0

        RELEASE = false

        REGISTRY = "NAME_OF_REGISTRY"
        
        // This file should be cotains only name and vesion of image e.g: tomcat:latest
        REF_IMAGE = readFile "./nexus-image-reference.txt"
    }


     // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        timeout(time: 15, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
    }

    stages {

        stage("Prepare environment for master deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*master)/ }
            }
            steps {
                script {
                    // Arbitrary Groovy Script executions can do in script tags
                    env.PROJECT_NAMESPACE = "${NAMESPACE_PREFIX}-test"
                    env.NODE_ENV = "test"
                    env.SPRING_PROFILES_ACTIVE = "openshift-test"
                    env.RELEASE = true
                }
            }
        }

        stage("Prepare environment for develop deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*develop|.*feature.*)/ }
            }
            steps {
                script {
                    // Arbitrary Groovy Script executions can do in script tags
                    env.PROJECT_NAMESPACE = "${NAMESPACE_PREFIX}-dev"
                    env.NODE_ENV = "dev"
                    env.SPRING_PROFILES_ACTIVE = "openshift-dev"
                }
            }
        }

        stage("Ansible") {
            agent {
                node {
                    label "jenkins-slave-ansible-devel"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*master|.*develop|.*feature.*)/ }
            }
            stages { 
                stage("Ansible Galaxy") {
                    steps {
                        echo '### Ansible Galaxy Installing Requirements ###'
                        sh "ansible-galaxy install -r .openshift-applier/requirements.yml --roles-path=.openshift-applier/roles"
                    }
                }
                stage("Apply Inventory using Ansible-Playbook") {
                    steps {
                        echo '### Apply Inventory using Ansible-Playbook ###'
                        sh "ansible-playbook .openshift-applier/apply.yml -i .openshift-applier/inventory/ -e 'include_tags=build,${NODE_ENV}'"
                    }
                }
            }
        }

        stage("automation for openshift") {
            agent {
                node {
                    label "master"
                }
            }

            stages {

                stage("Import Image From Nexus") {
                    
                    sh '''
                        oc project ${PROJECT_NAMESPACE}
                        oc import-image ${PROJECT_NAMESPACE}/${REF_IMAGE} --from=${REGISTRY}/${REF_IMAGE} --confirm
                    '''
                }

                stage("Openshift Deployment") {
                     when {
                        allOf{
                            expression { GIT_BRANCH ==~ /(.*master|.*develop|.*feature.*)/ }
                            expression { currentBuild.result != 'UNSTABLE' }
                        }
                    }
                    steps {
                        echo '### tag image for namespace ###'
                        
                        sh  '''
                            oc project ${PROJECT_NAMESPACE}
                            oc tag ${PIPELINES_NAMESPACE}/${APP_NAME}:${JENKINS_TAG} ${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                        '''
                        
                        echo '### set env vars and image for deployment ###'

                        sh '''
                            oc set env dc ${APP_NAME} NODE_ENV=${NODE_ENV} SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE}
                            oc set image dc/${APP_NAME} ${APP_NAME}=docker-registry.default.svc:5000/${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                            oc label --overwrite dc ${APP_NAME} stage=${NODE_ENV}
                            oc patch dc ${APP_NAME} -p "{\\"spec\\":{\\"template\\":{\\"metadata\\":{\\"labels\\":{\\"version\\":\\"${VERSION}\\",\\"release\\":\\"${RELEASE}\\",\\"stage\\":\\"${NODE_ENV}\\",\\"git-commit\\":\\"${GIT_COMMIT}\\",\\"jenkins-build\\":\\"${JENKINS_TAG}\\"}}}}}"
                            oc rollout latest dc/${APP_NAME}
                        '''
                        echo '### Verify OCP Deployment ###'
                        openshiftVerifyDeployment depCfg: env.APP_NAME,
                            namespace: env.PROJECT_NAMESPACE,
                            replicaCount: '1',
                            verbose: 'false',
                            verifyReplicaCount: 'true',
                            waitTime: '',
                            waitUnit: 'sec'
                    }
                }
            }
        }
    }
}