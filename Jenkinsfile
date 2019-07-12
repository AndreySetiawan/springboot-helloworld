def version, mvnCmd = "mvn -s templates/cicd-settings-nexus3.xml"
      pipeline
      {
       agent any
        tools
        {
            maven 'M3'
        }

        stages
        {
          stage('Build App')
          {
            steps
             {
              git branch: 'openshift-aws', url: 'https://github.com/AndreySetiawan/springboot-helloworld.git'
              script {
                  def pom = readMavenPom file: 'pom.xml'
                  version = pom.version
              }
              sh "mvn install -DskipTests=true"
            }
          }
          stage('Test')
          {
            steps
            {
                  echo "Test Stage"
              sh "${mvnCmd} test -Dspring.profiles.active=test"
              //step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            }
          }
		  /*
          stage('Code Analysis')
          {
            steps
             {
              script
              {
                      sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000  -DskipTests=true"
              }
            }
          }
          stage('Archive App') {
            steps {
              sh "${mvnCmd} deploy -DskipTests=true -P nexus3"
            }
          }*/

          stage('Create Image Builder') {

            when {
              expression {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    return !openshift.selector("bc", "myproject").exists();
                  }
                }
              }
            }
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    openshift.newBuild("--name=myproject", "--image-stream=redhat-openjdk18-openshift:latest", "--binary=true")
                  }
                }
              }
            }
          }
          stage('Build Image') {
            steps {
              sh "rm -rf ocp && mkdir -p ocp/deployments"
              sh "pwd && ls -la target "
              sh "cp target/myproject-*.jar ocp/deployments"

              script {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    openshift.selector("bc", "myproject").startBuild("--from-dir=./ocp","--follow", "--wait=true")
                  }
                }
              }
            }
          }
          stage('Create DEV') {
            when {
              expression {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    return !openshift.selector('dc', 'myproject').exists()
                  }
                }
              }
            }
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    def app = openshift.newApp("myproject:latest")
                    app.narrow("svc").expose();

                    //http://localhost:8080/actuator/health
                    openshift.set("probe dc/myproject --readiness --get-url=http://:8091/actuator/health --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
                    openshift.set("probe dc/myproject --liveness  --get-url=http://:8091/actuator/health --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")

                    def dc = openshift.selector("dc", "myproject")
                    while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                        sleep 10
                    }
                    openshift.set("triggers", "dc/myproject", "--manual")
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
                    openshift.selector("dc", "myproject").rollout().latest();
                  }
                }
              }
            }
          }
          stage('Promote to STAGE?') {
            steps {
              script {
                openshift.withCluster() {
                  openshift.tag("${env.DEV_PROJECT}/bookstore:latest", "${env.STAGE_PROJECT}/myproject:${version}")
                }
              }
            }
          }
          stage('Deploy STAGE') {
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject(env.STAGE_PROJECT) {
                    if (openshift.selector('dc', 'myproject').exists()) {
                      openshift.selector('dc', 'myproject').delete()
                      openshift.selector('svc', 'myproject').delete()
                      openshift.selector('route', 'myproject').delete()
                    }

                    openshift.newApp("myproject:${version}").narrow("svc").expose()
                    openshift.set("probe dc/myproject --readiness --get-url=http://:8091/actuator/health --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
                    openshift.set("probe dc/myproject --liveness  --get-url=http://:8091/actuator/health --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")
                  }
                }
              }
            }
          }
        }
      }