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
        stash name:"jar", includes:"target/demo-0.0.1-SNAPSHOT.jar"
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
        sh "cp target/demo-0.0.1-SNAPSHOT.jar target/app.jar"
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("bc", "car-service").startBuild("--from-file=target/app.jar", "--wait=true")
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
              openshift.selector("dc", "car-service").rollout().latest();
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
        timeout(time:15, unit:'MINUTES') {
            input message: "Promote to STAGE?", ok: "Promote"
        }
        script {
          openshift.withCluster() {
            if (env.ENABLE_QUAY.toBoolean()) {
              withCredentials([usernamePassword(credentialsId: "${openshift.project()}-quay-cicd-secret", usernameVariable: "QUAY_USER", passwordVariable: "QUAY_PWD")]) {
                sh "skopeo copy docker://quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:latest docker://quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:stage --src-creds \"$QUAY_USER:$QUAY_PWD\" --dest-creds \"$QUAY_USER:$QUAY_PWD\" --src-tls-verify=false --dest-tls-verify=false"
              }
            } else {
              openshift.tag("${env.DEV_PROJECT}/tasks:latest", "${env.STAGE_PROJECT}/tasks:stage")
            }
          }
        }
      }
    }
    stage('Deploy STAGE') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.STAGE_PROJECT) {
              openshift.selector("dc", "car-service").rollout().latest();
            }
          }
        }
      }
    }
  }
}
