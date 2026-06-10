pipeline {
    agent any

    tools {
        maven 'M3'
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    parameters {
        choice(name: 'DEPLOY_TARGET', choices: ['CloudHub2', 'RuntimeFabric', 'Standalone-ARM', 'Standalone-Copy'], description: 'Deployment target')
        choice(name: 'SOURCE_TYPE', choices: ['git', 'artifact'], description: 'Build from Git repo or deploy a pre-built JAR')

        // Source: Git
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/marcogalanti/mulesoft-demo1.git', description: 'Git repository URL containing the Mule project (when source=git)')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Branch to checkout')

        // Source: Pre-built artifact
        string(name: 'ARTIFACT_PATH', defaultValue: '', description: 'Path or URL to pre-built JAR (when source=artifact)')

        // Common
        string(name: 'APP_NAME', defaultValue: 'mule-hello-app', description: 'Application name in Anypoint Platform')
        string(name: 'ENVIRONMENT', defaultValue: 'Sandbox', description: 'Anypoint environment (Production, Sandbox, etc.)')
        string(name: 'MULE_RUNTIME_VERSION', defaultValue: '4.11.2', description: 'Mule runtime version')

        // CloudHub 2.0
        choice(name: 'CH2_TARGET', choices: ['Cloudhub-US-East-1', 'Cloudhub-US-East-2', 'Cloudhub-US-West-1', 'Cloudhub-US-West-2', 'Cloudhub-CA-Central-1', 'Cloudhub-SA-East-1', 'Cloudhub-AP-Southeast-1', 'Cloudhub-AP-Southeast-2', 'Cloudhub-AP-Northeast-1', 'Cloudhub-EU-West-1', 'Cloudhub-EU-Central-1', 'Cloudhub-EU-West-2'], description: 'CloudHub 2.0: deployment region target')
        string(name: 'CH2_REPLICAS', defaultValue: '1', description: 'CloudHub 2.0: number of replicas')
        choice(name: 'CH2_VCORES', choices: ['0.1', '0.2', '0.5', '1', '1.5', '2', '2.5', '3', '3.5', '4'], description: 'CloudHub 2.0: vCore size')

        // Runtime Fabric
        string(name: 'RTF_TARGET', defaultValue: '', description: 'Runtime Fabric: target name')
        string(name: 'RTF_REPLICAS', defaultValue: '1', description: 'Runtime Fabric: number of replicas')

        // Standalone
        string(name: 'STANDALONE_TARGET_NAME', defaultValue: '', description: 'Standalone ARM: server/cluster name registered in Runtime Manager')
        choice(name: 'STANDALONE_TARGET_TYPE', choices: ['server', 'serverGroup', 'cluster'], description: 'Standalone ARM: target type')
        string(name: 'STANDALONE_HOST', defaultValue: '', description: 'Standalone Copy: SSH host (user@host)')
        string(name: 'STANDALONE_APPS_DIR', defaultValue: '/opt/mule/apps', description: 'Standalone Copy: remote apps directory path')
        string(name: 'STANDALONE_SSH_CREDENTIALS_ID', defaultValue: 'standalone-ssh-key', description: 'Standalone Copy: Jenkins SSH credentials ID')
    }

    environment {
        PATH               = "/opt/homebrew/bin:/opt/homebrew/Cellar/maven/3.9.11/libexec/bin:${env.PATH}"
        JAVA_HOME          = '/Applications/AnypointStudio.app/Contents/Eclipse/plugins/org.mule.tooling.jdk.macosx.aarch64_1.4.1/Contents/Home'
        ANYPOINT_CLIENT_ID     = credentials('anypoint-client-id')
        ANYPOINT_CLIENT_SECRET = credentials('anypoint-client-secret')
        ANYPOINT_ORG           = credentials('anypoint-org')
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    echo "Deploy Target: ${params.DEPLOY_TARGET}"
                    echo "Source Type: ${params.SOURCE_TYPE}"
                    echo "App Name: ${params.APP_NAME}"
                    echo "Environment: ${params.ENVIRONMENT}"
                    echo "Runtime Version: ${params.MULE_RUNTIME_VERSION}"
                }
            }
        }

        stage('Checkout') {
            when { expression { params.SOURCE_TYPE == 'git' } }
            steps {
                dir('mule-app') {
                    git url: params.GIT_REPO_URL, branch: params.GIT_BRANCH
                }
            }
        }

        stage('Fetch Artifact') {
            when { expression { params.SOURCE_TYPE == 'artifact' } }
            steps {
                script {
                    def artifactPath = params.ARTIFACT_PATH
                    if (artifactPath.startsWith('http://') || artifactPath.startsWith('https://')) {
                        sh "mkdir -p mule-app/target && curl -fSL -o mule-app/target/app.jar '${artifactPath}'"
                    } else {
                        sh "mkdir -p mule-app/target && cp '${artifactPath}' mule-app/target/app.jar"
                    }
                }
            }
        }

        stage('Build') {
            when { expression { params.SOURCE_TYPE == 'git' } }
            steps {
                dir('mule-app') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test') {
            when { expression { params.SOURCE_TYPE == 'git' } }
            steps {
                dir('mule-app') {
                    sh 'mvn test'
                }
            }
        }

        stage('Resolve Artifact') {
            steps {
                dir('mule-app') {
                    script {
                        if (params.SOURCE_TYPE == 'git') {
                            env.MULE_ARTIFACT = sh(
                                script: "ls target/*.jar | grep -v original | head -1",
                                returnStdout: true
                            ).trim()
                        } else {
                            env.MULE_ARTIFACT = 'target/app.jar'
                        }
                        echo "Artifact: ${env.MULE_ARTIFACT}"
                    }
                }
            }
        }

        stage('Publish on Exchange') {
            when { expression { params.SOURCE_TYPE == 'git' } }
            steps {
                dir('mule-app') {
                    sh 'mvn deploy -DskipTests'
                }
            }
        }

        stage('Deploy') {
            steps {
                dir('mule-app') {
                    script {
                        def mvnCmd = 'mvn deploy -DmuleDeploy -DskipTests'
                        mvnCmd += " -Danypoint.client_id=${env.ANYPOINT_CLIENT_ID}"
                        mvnCmd += " -Danypoint.client_secret=${env.ANYPOINT_CLIENT_SECRET}"
                        mvnCmd += " -Danypoint.org=${env.ANYPOINT_ORG}"
                        mvnCmd += " -Danypoint.environment=${params.ENVIRONMENT}"

                        switch (params.DEPLOY_TARGET) {
                            case 'CloudHub2':
                                mvnCmd += " -Dcloudhub2.applicationName=${params.APP_NAME}"
                                mvnCmd += " -Dcloudhub2.target=${params.CH2_TARGET}"
                                mvnCmd += " -Dcloudhub2.replicas=${params.CH2_REPLICAS}"
                                mvnCmd += " -Dcloudhub2.vCores=${params.CH2_VCORES}"
                                mvnCmd += " -Dcloudhub2.muleVersion=${params.MULE_RUNTIME_VERSION}"
                                break
                            case 'RuntimeFabric':
                                mvnCmd += " -Drtf.target=${params.RTF_TARGET}"
                                mvnCmd += " -Drtf.applicationName=${params.APP_NAME}"
                                mvnCmd += " -Drtf.replicas=${params.RTF_REPLICAS}"
                                mvnCmd += " -Drtf.muleVersion=${params.MULE_RUNTIME_VERSION}"
                                break
                            case 'Standalone-ARM':
                                mvnCmd += " -Dstandalone.applicationName=${params.APP_NAME}"
                                mvnCmd += " -Dstandalone.target=${params.STANDALONE_TARGET_NAME}"
                                mvnCmd += " -Dstandalone.targetType=${params.STANDALONE_TARGET_TYPE}"
                                mvnCmd += " -Dstandalone.muleVersion=${params.MULE_RUNTIME_VERSION}"
                                break
                            case 'Standalone-Copy':
                                echo "Skipping Maven deploy — using SCP"
                                return
                        }

                        sh mvnCmd
                    }
                }
            }
        }

        stage('Copy to Server') {
            when { expression { params.DEPLOY_TARGET == 'Standalone-Copy' } }
            steps {
                dir('mule-app') {
                    script {
                        sshagent(credentials: [params.STANDALONE_SSH_CREDENTIALS_ID]) {
                            sh """scp ${env.MULE_ARTIFACT} ${params.STANDALONE_HOST}:${params.STANDALONE_APPS_DIR}/"""
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Mule application '${params.APP_NAME}' deployed successfully to ${params.DEPLOY_TARGET}"
        }
        failure {
            echo "Deployment failed — check stage logs for details"
        }
    }
}