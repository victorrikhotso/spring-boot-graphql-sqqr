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
              // openshift.selector("dc", "car-service").rollout().latest();
              def bc = openshift.selector("bc", "car-service")

              // The build config will create a new build object automatically, but how do
              // we find it? The 'related(kind)' operation can create an appropriate Selector for us.
              def builds = bc.related('builds')

              // There are no guarantees in life, so let's interrupt these operations if they
              // take more than 10 minutes and fail this script.
              timeout(10) {

                  // We can use watch to execute a closure body each objects selected by a Selector
                  // change. The watch will only terminate when the body returns true.
                  builds.watch {
                      // Within the body, the variable 'it' is bound to the watched Selector(i.e. builds)
                      echo "So far, ${bc.name()} has created builds: ${it.names()}"

                      // End the watch only once a build object has been created.
                      return it.count() > 0
                  }

                  // But we can actually want to wait for the build to complete.
                  builds.watch {
                      if (it.count() == 0) return false

                      // A robust script should not assume that only one build has been created, so
                      // we will need to iterate through all builds.
                      def allDone = true
                      it.withEach {
                          // 'it' is now bound to a Selector selecting a single object for this iteration.
                          // Let's model it in Groovy to check its status.
                          def buildModel = it.object()
                          if (it.object().status.phase != "Complete") {
                              allDone = false
                          }
                      }

                      return allDone;}


                  // Uggh. That was actually a significant amount of code. Let's use untilEach(){}
                  // instead. It acts like watch, but only executes the closure body once
                  // a minimum number of objects meet the Selector's criteria only terminates
                  // once the body returns true for all selected objects.
                  builds.untilEach(1) { // We want a minimum of 1 build

                      // Unlike watch(), untilEach binds 'it' to a Selector for a single object.
                      // Thus, untilEach will only terminate when all selected objects satisfy this
                      // the condition established in the closure body (or until the timeout(10)
                      // interrupts the operation).

                      return it.object().status.phase == "Complete"
                  }

                  }

            def dc = openshift.selector('dc', "car-service")
            // this will wait until the desired replicas are available
            dc.rollout().status()
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
            if (env.ENABLE_QUAY.toBoolean()) {
              withCredentials([usernamePassword(credentialsId: "${openshift.project()}-quay-cicd-secret", usernameVariable: "QUAY_USER", passwordVariable: "QUAY_PWD")]) {
                sh "skopeo copy docker://quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:latest docker://quay.io/${QUAY_USERNAME}/${QUAY_REPOSITORY}:stage --src-creds \"$QUAY_USER:$QUAY_PWD\" --dest-creds \"$QUAY_USER:$QUAY_PWD\" --src-tls-verify=false --dest-tls-verify=false"
              }
            } else {
              openshift.tag("${env.DEV_PROJECT}/car-service:latest",
                            "${env.STAGE_PROJECT}/car-service:stage")
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
              openshift.selector("dc", "car-service").rollout().latest();}
          }
        }
      }
    }
  }
}
