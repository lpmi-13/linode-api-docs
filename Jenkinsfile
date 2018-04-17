// vim: ft=groovy ts=2 sw=4 et
def docs_version = 'UNKNOWN'
def post_tag_commits = ''
def repos = [master: 'linode-internal-apt-jessie-stg', develop: 'linode-internal-apt-jessie-dev', release: 'linode-internal-apt-jessie-testing']

environment {
    GA_ID = '{DEBCONF_CLOUD_GA_ID}'
    APP_ROOT = '{DEBCONF_APP_ROOT}'
    API_ROOT = '{DEBCONF_API_ROOT}'
    LOGIN_ROOT = '{DEBCONF_LOGIN_ROOT}'
    API_VERSION = 'v4' // used by docs - TODO can remove?
    BUILD_ENV = "${(env.BRANCH_NAME != 'master') ? env.BRANCH_NAME : '' }"
    // POST_TAG_COMMITS = "$post_tag_commits" - TODO can remove?
}

node {
    def image;

    stage('Checkout') {
        deleteDir()
        checkout scm
        sh "git fetch"
    }

    stage('Build Docker') {
        image = docker.build(env.BUILD_TAG.toLowerCase(), '.')
    }

    stage ('Apply Environment') {
        replace_to = ''
        if (env.BRANCH_NAME == 'development') {
            replace_to = 'https://api.dev.linode.com/v4'
        } else if (env.BRANCH_NAME ==~ /^release\/\d+\.\d+$/) { 
            replace_to = 'https://api.testing.linode.com/v4'
        } else {
            echo "Skipping environment-specific changes"
        }

        if (replace_to != '') {
            echo "Applying environment-specific changes"
            image.inside() { c ->
                sh "sed -i -- 's|https://api.linode.com/v4|${replace_to}' openapi.yaml"
            }
        }
    }

    stage ('OpenAPI Lint') {
        echo "Linting openapi.yaml"
        image.inside() { c ->
            sh "python3 openapi-linter.py openapi.yaml"
        }
    }

    stage('Package Docs') {
        image.inside() { c ->
            sh "./build-docs.sh"
            archive '*.deb,*.changes'
        }
    }

    if (env.BRANCH_NAME == 'master') {
        stage ('Apt-Repo') {
            sh "dupload --to linode-internal-apt-jessie-stg --nomail *.changes"
        }
    } else if (env.BRANCH_NAME == 'development') {
        stage('Apt-Repo') {
            sh "dupload --to linode-internal-apt-jessie-dev --nomail *.changes"
        }
    } else if (env.BRANCH_NAME ==~ /^release\/\d+\.\d+$/) { 
        stage('Apt-Repo') {
            sh "dupload --to linode-internal-apt-jessie-testing --nomail *.changes"
        }
    } else {
        echo "Branch '${env.BRANCH_NAME}' is not 'master', 'development', or a release branch.  Skipping package upload."
    }
}
