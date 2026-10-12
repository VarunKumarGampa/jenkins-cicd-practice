pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')               // kill the whole run if it hangs
        buildDiscarder(logRotator(numToKeepStr: '10'))   // keep only the last 10 builds
    }

    triggers {
        pollSCM('H/2 * * * *')                           // check Git for new commits every ~2 minutes
    }

    parameters {
        choice(name: 'FAIL_AT', choices: ['none', 'test', 'prod'],
               description: 'Simulate a failure at this stage')
    }

    stages {
        stage('Set Version') {
            steps {
                script {
                    def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE = "login-app:${sha}"
                }
                echo "Release version: ${env.IMAGE}"
            }
        }

        stage('Quality Gates') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'echo "Running unit tests"; sleep 5; echo "Unit tests passed"' }
                }
                stage('Code Scan') {
                    steps { sh 'echo "Running SonarQube scan"; sleep 5; echo "Quality gate passed"' }
                }
            }
        }

        stage('Build & Push Image') {
            steps { sh 'echo "Building and pushing $IMAGE (built ONCE)"' }
        }

        stage('Deploy to Dev') {
            steps { sh 'echo "Deploying $IMAGE to DEV"; echo "Smoke test passed"' }
        }

        stage('Deploy to Test') {
            steps {
                sh 'echo "Deploying $IMAGE to TEST"'
                sh '[ "$FAIL_AT" != "test" ] || { echo "Integration tests FAILED"; exit 1; }'
            }
        }

        stage('Deploy to Preprod') {
            steps { sh 'echo "Deploying $IMAGE to PREPROD"; echo "Load test passed"' }
        }

        stage('Approval') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    input message: "Change request approved? Deploy ${env.IMAGE} to PROD?", ok: 'Approve'
                }
            }
        }

        stage('Deploy to Prod') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                sh 'echo "Deploying $IMAGE to PROD"'
                sh '[ "$FAIL_AT" != "prod" ] || { echo "Health check FAILED"; exit 1; }'
            }
            post {
                failure { sh 'echo "ROLLBACK: restoring previous version in PROD"' }
            }
        }
    }

    post {
        success { echo "RELEASED ${env.IMAGE} to prod" }
        failure { echo "PIPELINE FAILED: notify team" }
        aborted { echo "Release aborted: rejected or approval timed out" }
    }
}
