// ═══════════════════════════════════════════════════════════════════════════
// Jenkinsfile — TechBuild Solutions / hello-world-2
// Repository: https://github.com/jagdishmodi/hello-world-2.git

// Pipeline: Checkout → Build → Test → Quality Analysis → Archive → Notify
// ═══════════════════════════════════════════════════════════════════════════
 
pipeline {
 
    // ── Docker agent for isolated, reproducible builds ─────────────────────
    agent {
        docker {
            image 'eclipse-temurin:17-jdk-alpine'
            args  '-v $HOME/.m2:/root/.m2'    // Cache Maven dependencies between builds
        }
    }
 
    // ── Environment variables ───────────────────────────────────────────────
    environment {
        APP_NAME     = 'hello-world-3'
        APP_VERSION  = "1.0.${env.BUILD_NUMBER}"
        MAVEN_OPTS   = '-Xmx1024m -XX:+TieredCompilation'
        SONAR_URL    = 'http://sonarqube:9000'
        ARTIFACT_DIR = 'target'
    }
 
    // ── Pipeline-wide options ───────────────────────────────────────────────
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        // ansiColor('xterm')
    }
 
    // ── Build on push to any branch; deploy only from main ─────────────────
    triggers {
        githubPush()    // Requires GitHub plugin — responds to webhook events
    }
 
    // ══════════════════════════════════════════════════════════════════════
    stages {
 
        // ── STAGE 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH} | Commit: ${env.GIT_COMMIT[0..7]}"
            }
        }
 
        // ── STAGE 2: Build ────────────────────────────────────────────────
        stage('Build') {
            tools { maven 'Maven-3.9.11' }
            steps {
                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }
            post {
                success { echo 'Compile successful — moving to Test stage.' }
                failure { echo 'Compile FAILED — check pom.xml and source errors.' }
            }
        }
 
        // ── STAGE 3: Test ─────────────────────────────────────────────────
        stage('Test') {
            tools { maven 'Maven-3.9.11' }
            steps {
                sh 'mvn test -B'
            }
            post {
                always {
                    // Publish JUnit test results regardless of pass/fail
                    junit(testResults: 'target/surefire-reports/**/*.xml',
                          allowEmptyResults: false)
                }
                unstable {
                    echo 'WARNING: Tests failed — build marked UNSTABLE.'
                    // Enforce 80% pass rate:
                    script {
                        def results = currentBuild.rawBuild.getAction(
                            hudson.tasks.test.AbstractTestResultAction.class)
                        if (results) {
                            def passRate = (results.totalCount - results.failCount) /
                                           results.totalCount * 100
                            if (passRate < 80) {
                                error("Test pass rate ${passRate.round(1)}% is below 80% threshold!")
                            }
                        }
                    }
                }
            }
        }
 
        // ── STAGE 4: Quality Analysis ─────────────────────────────────────
        stage('Quality Analysis') {
            tools { maven 'Maven-3.9.11' }
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
 
        // ── STAGE 6: Package & Archive ────────────────────────────────────
        stage('Package & Archive') {
            tools { maven 'Maven-3.9.11' }
            steps {
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"
                archiveArtifacts(artifacts: 'target/*.jar', fingerprint: true)
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}.jar"
            }
        }
 
        // ── STAGE 7: Publish to Nexus (main branch only) ─────────────────
        stage('Publish Artifact') {
            when { branch 'main' }
            tools { maven 'Maven-3.9.11' }
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
                        file:       "target/${env.APP_NAME}-${env.APP_VERSION}.jar",
                        type:       'jar'
                    ]]
                )
            }
        }
 
    }    // end stages
 
    // ── Post-build actions ─────────────────────────────────────────────────
    post {
        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"
            slackSend(
                channel: '#ci-notifications',
                color:   'good',
                message: "BUILD PASSED: ${env.APP_NAME} v${env.APP_VERSION} | ${env.BUILD_URL}"
            )
            emailext(
                to:       'aggarsahil3@gmail.com',
                subject:  "BUILD PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:     "Successful build for ${env.APP_NAME} v${env.APP_VERSION}\nURL: ${env.BUILD_URL}"
            )
        }
        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"
            slackSend(
                channel: '#ci-notifications',
                color:   'danger',
                message: "BUILD FAILED: ${env.APP_NAME} #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
            )
            emailext(
                to:       'aggarsahil3@gmail.com',
                subject:  "BUILD FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:     "Build ${env.BUILD_NUMBER} failed.\nConsole: ${env.BUILD_URL}console"
            )
        }
        unstable {
            slackSend(
                channel: '#ci-notifications',
                color:   'warning',
                message: "BUILD UNSTABLE: ${env.APP_NAME} #${env.BUILD_NUMBER} — test failures | ${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()    // Clean workspace after every build to free disk space
        }
    }
 
}    // end pipeline
