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

        pom = readMavenPom file: 'infra-spring/pom.xml'

        ARTIFACTID = pom.getArtifactId();
        VERSION = pom.getVersion();

        RELEASE = false
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
            stages{
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


    }

}