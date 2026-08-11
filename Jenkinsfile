// ═══════════════════════════════════════════════════════════════════════════
// Jenkinsfile — hello-world-3
// Pipeline: Checkout → Build → Test → Quality Analysis → Quality Gate
//           → Package & Archive → Publish Artifact
// Java: 21
// Maven: 3.9.11
// ═══════════════════════════════════════════════════════════════════════════

pipeline {

    // ── Docker agent with Java 21 + Maven 3.9.11 ─────────────────────────
    agent {
        docker {
            image 'maven:3.9.11-eclipse-temurin-21-alpine'

            // Mount Jenkins Maven repository and explicitly set HOME
            args '-v /var/lib/jenkins/.m2:/root/.m2'
        }
    }

    // ── Environment ──────────────────────────────────────────────────────
    environment {
        APP_NAME    = 'hello-world-3'
        APP_VERSION = "1.0.${env.BUILD_NUMBER}"

        // Explicit writable Maven repository
        MAVEN_OPTS = '-Xmx1024m -XX:+TieredCompilation -Dmaven.repo.local=/root/.m2/repository'

        ARTIFACT_DIR = 'target'
    }

    // ── Pipeline options ─────────────────────────────────────────────────
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    // ── GitHub push trigger ──────────────────────────────────────────────
    triggers {
        githubPush()
    }

    stages {

        // ════════════════════════════════════════════════════════════════
        // STAGE 1 — CHECKOUT
        // ════════════════════════════════════════════════════════════════
        stage('Checkout') {
            steps {
                checkout scm

                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT[0..7]}"
            }
        }

        // ════════════════════════════════════════════════════════════════
        // STAGE 2 — BUILD
        // ════════════════════════════════════════════════════════════════
        stage('Build') {
            steps {

                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"

                // Verify Java 21
                sh 'java -version'

                // Verify Maven
                sh 'mvn -version'

                // Build
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }

            post {
                success {
                    echo 'Compile successful — moving to Test stage.'
                }

                failure {
                    echo 'Compile FAILED — check pom.xml and source errors.'
                }
            }
        }

        // ════════════════════════════════════════════════════════════════
        // STAGE 3 — TEST
        // ════════════════════════════════════════════════════════════════
        stage('Test') {
            steps {
                sh 'mvn test -B'
            }

            post {
                always {
                    junit(
                        testResults: 'target/surefire-reports/**/*.xml',
                        allowEmptyResults: true
                    )
                }
            }
        }

        // ════════════════════════════════════════════════════════════════
        // STAGE 4 — QUALITY ANALYSIS
        // ════════════════════════════════════════════════════════════════
        stage('Quality Analysis') {
            steps {

                withSonarQubeEnv('SonarQube-Local') {

                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${env.APP_NAME} \
                          -Dsonar.projectName="TechBuild ${env.APP_NAME}" \
                          -Dsonar.projectVersion=${env.APP_VERSION} \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -B
                    """
                }
            }
        }

        // ════════════════════════════════════════════════════════════════
        // STAGE 5 — QUALITY GATE
        // ════════════════════════════════════════════════════════════════
        stage('Quality Gate') {

            agent none

            steps {

                timeout(time: 5, unit: 'MINUTES') {

                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ════════════════════════════════════════════════════════════════
        // STAGE 6 — PACKAGE & ARCHIVE
        // ════════════════════════════════════════════════════════════════
        stage('Package & Archive') {

            steps {

                sh """
                    mvn package \
                      -DskipTests \
                      -Drevision=${env.APP_VERSION} \
                      -B
                """

                archiveArtifacts(
                    artifacts: 'target/*.war,target/*.jar',
                    fingerprint: true,
                    allowEmptyArchive: false
                )

                echo "Artifact archived successfully."
            }
        }

        // ════════════════════════════════════════════════════════════════
        // STAGE 7 — PUBLISH ARTIFACT
        // ════════════════════════════════════════════════════════════════
        stage('Publish Artifact') {

            when {
                branch 'master'
            }

            steps {

                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: 'localhost:8081',
                    groupId: 'io.techbuild',
                    version: env.APP_VERSION,
                    repository: 'techbuild-releases',
                    credentialsId: 'nexus-creds',

                    artifacts: [[
                        artifactId: env.APP_NAME,
                        classifier: '',
                        file: "target/${env.APP_NAME}-${env.APP_VERSION}.war",
                        type: 'war'
                    ]]
                )
            }
        }
    }

    // ════════════════════════════════════════════════════════════════════
    // POST BUILD
    // ════════════════════════════════════════════════════════════════════

    post {

        success {

            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"

            slackSend(
                channel: '#ci-notifications',
                color: 'good',
                message: "BUILD PASSED: ${env.APP_NAME} v${env.APP_VERSION} | ${env.BUILD_URL}"
            )

            emailext(
                to: 'aggarsahil3@gmail.com',
                subject: "BUILD PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Successful build for ${env.APP_NAME} v${env.APP_VERSION}

URL:
${env.BUILD_URL}
"""
            )
        }

        failure {

            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"

            slackSend(
                channel: '#ci-notifications',
                color: 'danger',
                message: "BUILD FAILED: ${env.APP_NAME} #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
            )

            emailext(
                to: 'aggarsahil3@gmail.com',
                subject: "BUILD FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Build ${env.BUILD_NUMBER} failed.

Console:
${env.BUILD_URL}console
"""
            )
        }

        unstable {

            slackSend(
                channel: '#ci-notifications',
                color: 'warning',
                message: "BUILD UNSTABLE: ${env.APP_NAME} #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
            )
        }

        always {

            cleanWs()
        }
    }
}
