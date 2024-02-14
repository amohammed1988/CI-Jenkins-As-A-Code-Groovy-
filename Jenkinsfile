pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }
    
    environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '172.31.23.189'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        SNS_TOPIC_ARN = 'arn:aws:sns:us-east-2:405658577042:Flint-Notify'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
               withSonarQubeEnv("${SONARSERVER}") {
                   sh '''${scannerHome}/bin/sonar-scanner \
                   -Dsonar.projectKey=CI-Jenkins-As-A-Code-Groovy- \
                   -Dsonar.projectName=CI-Jenkins-As-A-Code-Groovy- \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
              }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
                // Publish to SNS
                script {
                    snsPublish(topicArn: "${SNS_TOPIC_ARN}", message: "Quality Gate passed for ${env.JOB_NAME}")
                }
            }
        }

        stage("UploadArtifact") {
            steps {
                // Upload artifact to Nexus
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                        [artifactId: 'CI-Jenkins-As-A-Code-Groovy-',
                         classifier: '',
                         file: 'target/vprofile-v2.war',
                         type: 'war']
                    ]
                )
                // Log to CloudWatch
                script {
                    awsCloudWatchLog(logGroup: 'JenkinsPipeline', logStream: 'BuildLog', message: "Artifact uploaded for ${env.JOB_NAME}")
                }
            }
        }
    }
}

def snsPublish(params) {
    def sns = new AmazonSNSClient()
    sns.publish(new PublishRequest().withTopicArn(params.topicArn).withMessage(params.message))
}

def awsCloudWatchLog(params) {
    def cloudWatchLogs = new AWSLogsClient()
    cloudWatchLogs.createLogStream(new CreateLogStreamRequest().withLogGroupName(params.logGroup).withLogStreamName(params.logStream))
    cloudWatchLogs.putLogEvents(new PutLogEventsRequest().withLogGroupName(params.logGroup).withLogStreamName(params.logStream).withLogEvents([new InputLogEvent().withMessage(params.message)]))
}
