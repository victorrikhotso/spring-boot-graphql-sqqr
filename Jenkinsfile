def mvnCmd = "mvn -s /nexus3/settings.xml"


pipeline {
    agent {
        label 'maven'
    }
    stages {
        stage('Build App') {
            steps {
                git url: "https://github.com/victorrikhotso/spring-boot-graphql-sqqr.git"
                sh "${mvnCmd} install -DskipTests=true"
                stash name: "jar", includes: "target/demo-0.0.1-SNAPSHOT.jar"
            }
        }
        stage('Test') {
            steps {
                sh "${mvnCmd} test"
                step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            }
        }
        stage('Code Analysis') {
            steps {
                script {
                    sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
                }
            }
        }
        stage('Archive App') {
            steps {
                sh "${mvnCmd} deploy -DskipTests=true"
            }
        }
        stage('Build Image') {
            steps {
                sh "cp target/demo-0.0.1-SNAPSHOT.jar target/demo.jar"
                script {
                    openshift.withCluster() {
                        openshift.withProject(env.DEV_PROJECT) {
                            openshift.selector(
                                "bc", "car-service").startBuild("--from-file=target/demo.jar")
                        }
                    }
                }
            }
        }
        stage('Deploy DEV') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(env.DEV_PROJECT) {
                            def latestDeploymentVersion = openshift.selector("dc", "car-service").object().status.latestVersion
                            def rc = openshift.selector("rc", "car-service-${latestDeploymentVersion}")
                            rc.untilEach(1){
                                def rcMap = it.object()
                                return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
                            }
                        }
                    }
                }
            }
        }
        stage('Promote to STAGE?') {
            agent {
                label 'skopeo'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Promote to STAGE?", ok: "Promote"
                }
                script {
                    openshift.withCluster() {

                            openshift.tag("${env.DEV_PROJECT}/car-service:latest",
                                          "${env.STAGE_PROJECT}/car-service:stage")
                    }
                }
            }
        }
        stage('Deploy STAGE') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(env.STAGE_PROJECT) {
                            openshift.selector(
                                "dc", "car-service").rollout().latest()
                        }
                    }
                }
            }
        }
    }
}
