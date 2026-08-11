// ═══════════════════════════════════════════════════════════════════════════
// Jenkinsfile — TechBuild Solutions / hello-world-2
// Repository: https://github.com/jagdishmodi/hello-world-2.git
//
// Pipeline: Checkout → Build → Test → Quality Analysis → Quality Gate
//           → Package & Archive → Publish Artifact
// ═══════════════════════════════════════════════════════════════════════════

pipeline {

    // ── Docker agent for isolated, reproducible builds ─────────────────────
    agent {
        docker {
            image 'maven:3.9.11-eclipse-temurin-17'
        }
    }

    // ── Environment variables ───────────────────────────────────────────────
    environment {
        APP_NAME     = 'hello-world-3'
        APP_VERSION  = "1.0.${env.BUILD_NUMBER}"

        // IMPORTANT:
        // Store Maven dependencies inside the Jenkins workspace.
        // This avoids the /root/.m2 permission problem inside the container.
        MAVEN_OPTS   = '-Xmx1024m -XX:+TieredCompilation -Dmaven.repo.local=.m2/repository'

        SONAR_URL    = 'http://sonarqube:9000'
        ARTIFACT_DIR = 'target'
    }

    // ── Pipeline-wide options ───────────────────────────────────────────────
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    // ── Build on GitHub push ────────────────────────────────────────────────
    triggers {
        githubPush()
    }

    // ══════════════════════════════════════════════════════════════════════
    stages {

        // ── STAGE 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm

                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT[0..7]}"
            }
        }

        // ── STAGE 2: Build ────────────────────────────────────────────────
        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"

                sh 'java -version'
                sh 'mvn -version'

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

        // ── STAGE 3: Test ─────────────────────────────────────────────────
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

                unstable {
                    echo 'WARNING: Tests failed — build marked UNSTABLE.'

                    script {
                        def results = currentBuild.rawBuild.getAction(
                            hudson.tasks.test.AbstractTestResultAction.class
                        )

                        if (results && results.totalCount > 0) {
                            def passRate =
                                (results.totalCount - results.failCount) /
                                results.totalCount * 100

                            if (passRate < 80) {
                                error(
                                    "Test pass rate ${passRate.round(1)}% is below 80% threshold!"
                                )
                            }
                        }
                    }
                }
            }
        }

        // ── STAGE 4: Quality Analysis ─────────────────────────────────────
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

        // ── STAGE 5: Quality Gate ─────────────────────────────────────────
        stage('Quality Gate') {
            agent none

            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── STAGE 6: Package & Archive ───────────────────────────────────
        stage('Package & Archive') {
            steps {
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"

                // Your pom.xml uses <packaging>war</packaging>,
                // therefore archive the WAR instead of a JAR.
                archiveArtifacts(
                    artifacts: 'target/*.war',
                    fingerprint: true
                )

                echo "Artifact archived successfully."
            }
        }

        // ── STAGE 7: Publish to Nexus (main branch only) ────────────────
        stage('Publish Artifact') {
            when {
                branch 'main'
            }

            steps {
                nexusArtifactUploader(
                    nexusVersion:  'nexus3',
                    protocol:      'http',
                    nexusUrl:      'localhost:8081',
                    groupId:       'io.techbuild',
                    version:       env.APP_VERSION,
                    repository:    'techbuild-releases',
                    credentialsId: 'nexus-creds',

                    artifacts: [[
                        artifactId: env.APP_NAME,
                        classifier: '',
                        file:       "target/${env.APP_NAME}-${env.APP_VERSION}.war",
                        type:       'war'
                    ]]
                )
            }
        }

    } // end stages

    // ── Post-build actions ─────────────────────────────────────────────────
    post {

        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"

            // Slack/email removed temporarily because they are not configured
            // on this Jenkins instance.
        }

        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"

            // Slack/email removed temporarily because they are not configured
            // on this Jenkins instance.
        }

        unstable {
            echo "PIPELINE UNSTABLE — ${env.APP_NAME} #${env.BUILD_NUMBER}"
        }

        always {
            cleanWs()
        }
    }
}
