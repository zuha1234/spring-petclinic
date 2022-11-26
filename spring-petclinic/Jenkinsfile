// Instead of annotating an unnecessary import statement, the symbol _ is annotated, according to the annotation pattern.
@Library('adop-pluggable-scm-jenkinsfile') _

def repoName = "spring-petclinic"
def regRepo = "adop-cartridge-java-regression-tests"

pipeline {
    agent { label 'java8' }
    environment{
        // Sonar Project Info - You must replace these vars with the Project Name and Key from the ADOP Portal!
        SONAR_PROJECT_NAME = 'Simple Java project analyzed with the SonarQube Runner'
        SONAR_PROJECT_KEY = 'java-sonar-runner-simple'
        ENVIRONMENT_NAME = 'CI'
        }
    stages{
        stage("Reference Application Build"){
            steps{
                echo 'This is a reference Java Application pipeline.'
                deleteDir()
                checkout scmGet("${SCM_URL}", "${SCM_NAMESPACE}", "${repoName}", "${SCM_CREDENTIAL_ID}", 'master')
                sh "./mvnw clean install -DskipTests"
            }
        }
        stage("Reference Application Unit Tests"){
            steps{
                echo 'This job runs unit tests on Java Spring reference application.'
                sh "./mvnw test"
            }  
        }
        stage("Reference Application Code Analysis"){
            steps{
                echo 'This job runs code quality analysis for Java reference application using SonarQube.'
                echo "Checking the Build number $BUILD_NUMBER"
                script{
                    // requires SonarQube Scanner 2.8+
                    scannerHome = tool 'ADOP SonarScanner'
                    withSonarQubeEnv('ADOP Sonar') {
                        echo "SONAR_PROJECT_NAME $env.SONAR_PROJECT_NAME"
                        echo "SONAR_PROJECT_KEY $env.SONAR_PROJECT_KEY"
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectName=\"${env.SONAR_PROJECT_NAME}\" -Dsonar.projectKey=\"${env.SONAR_PROJECT_KEY}\" -Dsonar.projectVersion=1.0.$BUILD_NUMBER -Dsonar.sources=src/main/java -Dsonar.language=java -Dsonar.sourceEncoding=UTF-8 -Dsonar.scm.enabled=false -Dsonar.jacoco.reportPath=target/jacoco.exec"
                    }
                }
            }
        }
        stage("Reference Application Deploy"){
            steps{
                echo 'This job deploys the java reference application to the CI environment.'
                sh '''
                    set -x
                    export SERVICE_NAME="$(echo ${PROJECT_NAME} | tr \'/\' \'_\')_${ENVIRONMENT_NAME}"
                    docker cp ${WORKSPACE}/target/petclinic.war  ${SERVICE_NAME}:/usr/local/tomcat/webapps/
                    docker restart ${SERVICE_NAME}
                    COUNT=1
                    echo "Count : ${COUNT}"
                    while ! curl -q http://${SERVICE_NAME}:8080/petclinic -o /dev/null
                        do
                            if [ ${COUNT} -gt 10 ]; then
                                echo "Docker build failed even after ${COUNT}. Please investigate."
                                exit 1
                            fi
                            echo "Application is not up yet. Retrying ..Attempt (${COUNT})"
                            sleep 5
                            COUNT=$((COUNT+1))
                        done
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "Environment URL (replace PUBLIC_IP with your public ip address where you access jenkins from) : http://${SERVICE_NAME}.PUBLIC_IP.xip.io/petclinic"
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    set -x
                '''
                stash includes: '**/**', name: 'build-artefacts'
            }
        }
        stage("Reference Application Regression Tests"){
            steps{
                echo 'This job runs regression tests on deployed java application.'
                checkout scmGet("${SCM_URL}", "${SCM_NAMESPACE}", "${regRepo}", "${SCM_CREDENTIAL_ID}", 'master')
                withMaven(maven: 'ADOP Maven') {
                    sh '''
                        export SERVICE_NAME="$(echo ${PROJECT_NAME} | tr \'/\' \'_\')_${ENVIRONMENT_NAME}"
                        echo "SERVICE_NAME=${SERVICE_NAME}" > env.properties
                        echo "Running automation tests"
                        echo "Setting values for container, project and app names"
                        CONTAINER_NAME="owasp_zap-"${SERVICE_NAME}${BUILD_NUMBER}
                        
                        APP_IP=$( docker inspect --format \'{{ .NetworkSettings.Networks.\'"$DOCKER_NETWORK_NAME"\'.IPAddress }}\' ${SERVICE_NAME} )
                        APP_URL=http://${APP_IP}:8080/petclinic
                        ZAP_PORT="9090"
                    
                        echo CONTAINER_NAME=$CONTAINER_NAME >> env.properties
                        echo APP_URL=$APP_URL >> env.properties
                        echo ZAP_PORT=$ZAP_PORT >> env.properties
                    
                        echo "Starting OWASP ZAP Intercepting Proxy"
                        JOB_WORKSPACE_PATH="/var/lib/docker/volumes/jenkins_slave_home/_data/${PROJECT_NAME}/Reference_Application_Regression_Tests"
                        echo JOB_WORKSPACE_PATH=$JOB_WORKSPACE_PATH >> env.properties
                        mkdir -p ${JOB_WORKSPACE_PATH}/owasp_zap_proxy/test-results
                        docker run -it -d --net=$DOCKER_NETWORK_NAME -v ${JOB_WORKSPACE_PATH}/owasp_zap_proxy/test-results/:/opt/zaproxy/test-results/ -e affinity:container==jenkins-slave --name ${CONTAINER_NAME} -P nhantd/owasp_zap start zap-test

                        ZAP_IP=$( docker inspect --format \'{{ .NetworkSettings.Networks.\'"$DOCKER_NETWORK_NAME"\'.IPAddress }}\' ${CONTAINER_NAME} )
                        echo "ZAP_IP =  $ZAP_IP"
                        echo ZAP_IP=$ZAP_IP >> env.properties
                        echo ZAP_ENABLED="true" >> env.properties
                        echo "Running Selenium tests through maven."
                        
                        mvn clean -B test -DPETCLINIC_URL=${APP_URL} -DZAP_IP=${ZAP_IP} -DZAP_PORT=${ZAP_PORT} -DZAP_ENABLED=${ZAP_ENABLED}

                        echo "Stopping OWASP ZAP Proxy and generating report."
                        echo "Container ${CONTAINER_NAME}"
                        docker stop ${CONTAINER_NAME}
                        docker rm ${CONTAINER_NAME}
                    
                        docker run -i --net=$DOCKER_NETWORK_NAME -v ${JOB_WORKSPACE_PATH}/owasp_zap_proxy/test-results/:/opt/zaproxy/test-results/ -e affinity:container==jenkins-slave --name ${CONTAINER_NAME} -P nhantd/owasp_zap stop zap-test
                        docker cp ${CONTAINER_NAME}:/opt/zaproxy/test-results/zap-test-report.html .
                        sleep 10s
                        docker rm ${CONTAINER_NAME}
                    '''
                }
                cucumber fileIncludePattern: '', sortingMethod: 'ALPHABETICAL'
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'zap-test-report.html', reportName: 'ZAP security test report', reportTitles: ''])
            }
        }
        stage("Reference Application Performance Tests"){
            steps{
                echo 'This job run the Jmeter test for the java reference application.'
                unstash 'build-artefacts'
                sh '''
                    export SERVICE_NAME="$(echo ${PROJECT_NAME} | tr '/' '_')_${ENVIRONMENT_NAME}"
                    if ! grep -e apache-jmeter-2.13.tgz ; then
                        wget https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-2.13.tgz
                    fi
                    tar -xf apache-jmeter-2.13.tgz
                    echo 'Changing user defined parameters for jmx file'
                    sed -i 's/PETCLINIC_HOST_VALUE/'"${SERVICE_NAME}"'/g' ${WORKSPACE}/src/test/jmeter/petclinic_test_plan.jmx
                    sed -i 's/PETCLINIC_PORT_VALUE/8080/g' ${WORKSPACE}/src/test/jmeter/petclinic_test_plan.jmx
                    sed -i 's/CONTEXT_WEB_VALUE/petclinic/g' ${WORKSPACE}/src/test/jmeter/petclinic_test_plan.jmx
                    sed -i 's/HTTPSampler.path"></HTTPSampler.path">petclinic</g' ${WORKSPACE}/src/test/jmeter/petclinic_test_plan.jmx
                '''
                withAnt(installation: 'ADOP Ant'){
                    sh '''
                        ant -buildfile ${WORKSPACE}/apache-jmeter-2.13/extras/build.xml -Dtestpath=$WORKSPACE/src/test/jmeter -Dtest=petclinic_test_plan
                        export SERVICE_NAME="$(echo ${PROJECT_NAME} | tr '/' '_')_${ENVIRONMENT_NAME}"
                        CONTAINER_IP=$(docker inspect --format '{{ .NetworkSettings.Networks.'"$DOCKER_NETWORK_NAME"'.IPAddress }}' ${SERVICE_NAME})
                        sed -i "s/###TOKEN_VALID_URL###/http:\\/\\/${CONTAINER_IP}:8080/g" ${WORKSPACE}/src/test/gatling/src/test/scala/default/RecordedSimulation.scala
                        sed -i "s/###TOKEN_RESPONSE_TIME###/10000/g" ${WORKSPACE}/src/test/gatling/src/test/scala/default/RecordedSimulation.scala
                        ./mvnw -f ${WORKSPACE}/src/test/gatling/pom.xml gatling:execute
                    '''
                }
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: "${WORKSPACE}/src/test/jmeter/", reportFiles: 'petclinic_test_plan.html', reportName: 'Jmeter Report', reportTitles: ''])
            }
        }
        stage("Reference Application Deploy ProdA"){
            steps{
                echo 'This job deploys the java reference application to the ProdA environment'
                timeout(time:5, unit:'MINUTES') {
                    input('! Deploy to Prod A Environment?')
                }
                sh '''
                    set -x
                    export SERVICE_NAME="$(echo ${PROJECT_NAME} | tr \'/\' \'_\')_PRODA"
                    docker cp ${WORKSPACE}/target/petclinic.war  ${SERVICE_NAME}:/usr/local/tomcat/webapps/
                    docker restart ${SERVICE_NAME}
                    COUNT=1
                    echo "Count : ${COUNT}"
                    while ! curl -q http://${SERVICE_NAME}:8080/petclinic -o /dev/null
                        do
                            if [ ${COUNT} -gt 10 ]; then
                                echo "Docker build failed even after ${COUNT}. Please investigate."
                                exit 1
                            fi
                            echo "Application is not up yet. Retrying ..Attempt (${COUNT})"
                            sleep 5
                            COUNT=$((COUNT+1))
                        done
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "Environment URL (replace PUBLIC_IP with your public ip address where you access jenkins from) : http://${SERVICE_NAME}.PUBLIC_IP.xip.io/petclinic"
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    set -x
                '''
            }
        }
        stage("Reference Application Deploy ProdB"){
            steps{
                echo 'This job deploys the java reference application to the ProdB environment'
                timeout(time:5, unit:'MINUTES') {
                    input('! Deploy to Prod B Environment?')
                }
                sh '''
                    set -x
                    export SERVICE_NAME="$(echo ${PROJECT_NAME} | tr \'/\' \'_\')_PRODB"
                    docker cp ${WORKSPACE}/target/petclinic.war  ${SERVICE_NAME}:/usr/local/tomcat/webapps/
                    docker restart ${SERVICE_NAME}
                    COUNT=1
                    echo "Count : ${COUNT}"
                    while ! curl -q http://${SERVICE_NAME}:8080/petclinic -o /dev/null
                        do
                            if [ ${COUNT} -gt 10 ]; then
                                echo "Docker build failed even after ${COUNT}. Please investigate."
                                exit 1
                            fi
                            echo "Application is not up yet. Retrying ..Attempt (${COUNT})"
                            sleep 5
                            COUNT=$((COUNT+1))
                        done
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "Environment URL (replace PUBLIC_IP with your public ip address where you access jenkins from) : http://${SERVICE_NAME}.PUBLIC_IP.xip.io/petclinic"
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    echo "=.=.=.=.=.=.=.=.=.=.=.=."
                    set -x
                '''
            }
        }   
    }
}