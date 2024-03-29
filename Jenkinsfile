throttle(['one-per-node-throttle']) {
    node('linux') {
        properties([
                parameters([
                        string(name: 'DOCKER_COMPOSE_BRANCH', defaultValue: 'master', description: ''),
                        string(name: 'SOURCE_CODE_BRANCH', defaultValue: 'master', description: ''),
                        booleanParam(name: 'RUN_WITH_SPINNAKER_MOCK', defaultValue: true,
                                description: 'Running tests with Spinnaker mock')
                ]),
          [
            $class: 'ThrottleJobProperty',
            categories: ['my_category'],
            limitOneJobWithMatchingParams: false,
            maxConcurrentPerNode: 1,
            maxConcurrentTotal: 0,
            paramsToUseForLimit: '',
            throttleEnabled: true,
            throttleOption: 'category'
          ],
        ])
        def servicePorts = [8680, 8280, 8880, 8780, 8080, 8980] as int[]
        def services = [
                'beacon-configuration-service/',
                'beacon-equipment-service/',
                'beacon-oe3-service/',
                'beacon-tug-endpoints-service/',
                'beacon-tc-api-service/',
                'beacon-ems-service/beacon-ems-component/'
        ] as String[]
            environment {
                JAVA_VM_OPTIONS = "--add-opens java.base/jdk.internal.reflect=ALL-UNNAMED " +
                        "--add-opens java.base/jdk.internal.loader=ALL-UNNAMED " +
                        "--add-opens java.base/jdk.internal.module=ALL-UNNAMED  " +
                        "--add-opens java.base/java.lang.module=ALL-UNNAMED"
                JAVA_PATH = "/usr/lib/java-1.11.0-28"
            }

                stage('1. Pulling Code') {
                    options {
                        timeout(time: 5, unit: 'MINUTES')
                    }
                    steps {
                        deleteDir()
                        dropContainers()
                        downloadAllDependencies()
                    }
                }

                stage('2. Building') {
                    options {
                        timeout(time: 5, unit: 'MINUTES')
                    }
                    steps {
                        printStepName("BUILDING THE BEACON SERVER")
                        dir('source-code') {
                            withEnv(["JAVA_HOME=${JAVA_PATH}"]) {
                                sh 'mvn -q clean -U install'
                            }
                        }

                    }
                }

                stage('3. Starting Services') {
                    options {
                        timeout(time: 10, unit: 'MINUTES')
                    }
                    steps {
                        withEnv(["JAVA_HOME=${JAVA_PATH}"]) {
                            freeServicePorts(servicePorts)
                            startServices(services, servicePorts)
                        }
                    }
                }

                stage('4. Running Tests') {
                    options {
                        timeout(time: 30, unit: 'MINUTES')
                    }
                    steps {
                        printStepName("RUNNING THE BEACON QA TESTS")
                        dir('test') {
                            withEnv(["JAVA_HOME=$JAVA_PATH"]) {
                                sh "mvn clean -Dtestsuite=smoke install " +
                                        "-Duse-spinnaker-mock=${params.RUN_WITH_SPINNAKER_MOCK}"
                            }
                        }
                    }
                }

            }
            post {
                always {
                    echo 'One way or another, I have finished'
                    updateStatusAtBitbucket()
                    createAllureReport()
                    step([$class: 'Publisher', reportFilenamePattern: '**/testng-results.xml'])
                    perfReport filterRegex: '', sourceDataFiles: 'test/beacon-func-tests/logs/testResults.jtl'
                }
                failure {
                    collectLogsFromServices()
                    archiveArtifacts 'test/beacon-func-tests/logs/wire-mock.log'
                }
                cleanup {
                    freeServicePorts(servicePorts)
                    clearWorkspace()
                    deleteDir()
                }
            }

    }
}
def clearWorkspace() {
    def out = sh script: printWithNoTrace("sudo netstat -tulpn | grep 9092"), returnStatus: true
    if (out == 0) {
        cleanupKafkaTopics()
        removeAllContainersWithVolumes()
    }
}

def freeServicePorts(int[] ports) {
    echo("---TERMINATING PROCESSES ON BEACON SERVER PORTS---")
    for (int i = 0; i < ports.length; i++) {
        sh printWithNoTrace("sudo fuser -k ${ports[i]}/tcp || true")
    }
    echo "All PORTS ARE FREE"
}

def startServices(String[] services, int[] ports) {
    printStepName("STARTING BEACON SERVICES")
    for (int i = 0; i < services.length; i++) {
        def jarFile = null
        def target = "source-code/" + services[i] + "target/"
        dir(target) {
            jarFile = sh(script: "ls *.jar", returnStdout: true).trim()
        }
        sh "${JAVA_PATH}/bin/java -jar  -Xmx1536m $JAVA_VM_OPTIONS ${target}${jarFile} &"
        checkIfServiceIsUp(ports[i])
    }
}

def dropContainers() {
    echo "--DROP DOCKER CONTAINERS IF EXISTS--"
    sh script: "docker ps -a"
    sh printWithNoTrace("docker ps -a | awk '{ print \$1, \$2}' | grep '..tcb/infra..' | awk '{print \$1 }' | xargs -I{} docker rm -fv {}")
    echo "--TCB DOCKER CONTAINERS SHOULD BE DELETED. CHECKING:--"
    sh script: "docker ps -a"
}

def cleanupKafkaTopics() {
    echo "--CLEANING UP KAFKA TOPICS--"
    sh printWithNoTrace('docker exec -i -d -e COMPOSE_INTERACTIVE_NO_CLI=1 tcb-kafka sh ' +
            '-c "cd /opt/kafka/bin && kafka-topics.sh --delete --zookeeper tcb-zookeeper:2181 --topic tw.dev.*"')
    sh printWithNoTrace('sleep 30')
    echo "--KAFKA TOPICS WERE CLEANED--"
}

def removeAllContainersWithVolumes() {
    echo "--STOPPING DOCKER COMPOSE--"
    dir('compose') {
        timeout(time: 5, unit: 'MINUTES') {
            waitUntil {
                script {
                    def out = sh script: printWithNoTrace("docker-compose --no-ansi -f docker-compose-infra-smoke-test-job.yml down --remove-orphans"),
                            returnStatus: true
                    return (out == 0)
                }
            }
        }
        echo "--DOCKER WAS STOPPED--"
        echo "--REMOVING DOCKER CONTAINERS AND IMAGES--"
        timeout(time: 5, unit: 'MINUTES') {
            waitUntil {
                script {
                    def out = sh script: printWithNoTrace("docker system prune --all --force --volumes"),
                            returnStatus: true
                    return (out == 0)
                }
            }
        }
        echo "--CONTAINERS VOLUMES WERE REMOVED--"
    }
}

def downloadAllDependencies() {
    printStepName("DOWNLOADING BEACON SERVER REPOSITORY")
    dir('source-code') {
        git(
                url: 'ssh://git@bitbucket.tideworks.com:7999/tcb/tcb-beacon-server.git',
                credentialsId: '9f15dc89-7622-4f24-8362-6e5e1f683b5d',
                branch: "${params.SOURCE_CODE_BRANCH}",
                returnStdOut: true
        )
    }
    dir('test') {
        printStepName("DOWNLOADING BEACON QA TESTS REPOSITORY")
        git(
                url: 'ssh://git@bitbucket.tideworks.com:7999/tcb/beacon-qa-tests.git',
                credentialsId: '9f15dc89-7622-4f24-8362-6e5e1f683b5d',
                branch: "${BRANCH_NAME}",
                returnStdOut: true
        )
    }
    dir('compose') {
        printStepName("DOWNLOADING DOCKER REPOSITORY")
        git(
                url: 'ssh://git@bitbucket.tideworks.com:7999/tcb/tcb-docker-files.git',
                credentialsId: '9f15dc89-7622-4f24-8362-6e5e1f683b5d',
                branch: "${params.DOCKER_COMPOSE_BRANCH}",
                returnStdOut: true)
        pullDockerContainers()
    }
}

def checkIfServiceIsUp(Integer port) {
    echo "--CHECKING IF THE BEACON SERVICE ON THE PORT ${port} IS UP--"
    timeout(time: 5, unit: 'MINUTES') {
        waitUntil {
            sh printWithNoTrace("sleep 15")
            script {
                def out = sh script: printWithNoTrace("sudo netstat -tulpn | grep ${port}"), returnStatus: true
                return (out == 0)
            }
        }
    }
}

def pullDockerContainers() {
    printStepName("PULLING DOCKER CONTAINERS")
    timeout(time: 8, unit: 'MINUTES') {
        waitUntil {
            script {
                def out = sh script: printWithNoTrace("docker-compose -f docker-compose-infra-smoke-test-job.yml pull"),
                        returnStatus: true
                return (out == 0)
            }
        }
    }

    echo "--STARTING DOCKER CONTAINERS--"
    timeout(time: 8, unit: 'MINUTES') {
        waitUntil {
            script {
                def out = sh script: printWithNoTrace("docker-compose -f docker-compose-infra-smoke-test-job.yml up -d"),
                        returnStatus: true
                return (out == 0)
            }
        }
    }
    sh printWithNoTrace("docker container ps")
}

def printWithNoTrace(cmd) {
    return "#!/bin/sh -e\n" + cmd + " > /dev/null"
}

def printStepName(stepToPrint) {
    echo("""#########################################################################
# ${stepToPrint}
#########################################################################""")
}

def updateStatusAtBitbucket() {
    notifyBitbucket commitSha1: '',
            considerUnstableAsSuccess: false,
            credentialsId: '2f77ff3e-e7b4-49c4-9232-d57ddc3ed7ce',
            disableInprogressNotification: false,
            ignoreUnverifiedSSLPeer: false,
            includeBuildNumberInKey: false,
            prependParentProjectKey: false,
            projectKey: '',
            stashServerBaseUrl: 'https://bitbucket.tideworks.com'
}

def collectLogsFromServices() {
    printStepName("COLLECTING LOGS FROM THE SERVICES")
    sh "cp -fR logs services_logs ; zip -rq services_logs.zip services_logs"
    archiveArtifacts artifacts: '*.zip'
}

def createAllureReport() {
    script {
        allure([
                includeProperties: false,
                jdk              : '',
                properties       : [],
                reportBuildPolicy: 'ALWAYS',
                results          : [[path: '**/target/allure-results']]
        ])
    }
}
