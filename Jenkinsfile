def GIT_REPO = 'https://github.com/maheshacharya/java-sec-code'
def BRANCH = 'master'
def CONNECT = 'http://ip-172-31-32-101.us-east-2.compute.internal:8000'
def PROJECT = 'java-sec-code'
def STREAM = 'java-sec-code'
def CHECKERS = '--webapp-security --enable-callgraph-metrics'
def BLDCMD = 'mvn -B -f pom.xml install'
pipeline {
    agent {
        docker {
            image 'gsasig/magic-tool:latest'
        }
    }

    environment {
        POLARIS_HOME = '/tmp/polaris'
        AWS_HOST = "${sh(script:'curl -s http://169.254.169.254/latest/meta-data/local-hostname', returnStdout: true).trim()}"
        PUBLIC_HOST = "${sh(script:'curl -s http://169.254.169.254/latest/meta-data/public-hostname', returnStdout: true).trim()}"
    }

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
    }

    stages {
        stage('Build') {
            steps {
                echo "HOST = ${env.AWS_HOST}"

                // Get some code from a GitHub repository
                git 'https://github.com/maheshacharya/java-sec-code.git'

                // Run Maven on a Unix agent.
                sh "mvn clean package -DskipTests"
    
            }
        }
        stage('coverity') {
            steps {
                withCoverityEnvironment(coverityInstanceUrl: "$CONNECT", projectName: "$PROJECT", streamName: "$STREAM") {
                    sh """
                        rm -rf idir
                        $COVERITY_TOOL_HOME/bin/cov-configure --java 
                        $COVERITY_TOOL_HOME/bin/cov-build --dir idir $BLDCMD
                        $COVERITY_TOOL_HOME/bin/cov-analyze --dir idir --ticker-mode none --strip-path $WORKSPACE $CHECKERS
                        $COVERITY_TOOL_HOME/bin/cov-commit-defects --dir idir --ticker-mode none --url $COV_URL --stream $COV_STREAM --auth-key-file $COV_AUTH_KEY_PATH
                    """
                }
            }
        }
        stage('Run Java Sec Code with Seeker Agent') {
            steps {
                // DL seeker agent
                sh "wget --no-check-certificate --content-disposition \'http://$AWS_HOST:8088/rest/api/latest/installers/agents/binaries/JAVA?agentVersion=2022.2.0&flavor=JAR\'"
                // instrumented w/seeker agent
                sh 'nohup java -javaagent:./seeker-agent.jar  -Dseeker.project.key=java-sec-code -Dseeker.server.url=http://$AWS_HOST:8088 -jar target/java-sec-code-1.0.0.jar --server.port=9999 > jsc.out 2>&1 &'
                // DEBUG - no seeker agent
                //sh 'nohup java -jar target/java-sec-code-1.0.0.jar --server.port=9999 > jsc.out 2>&1 &'
                sh 'ps auxw | grep java-sec-code'
                sh 'sleep 60'
                sh 'curl -L http://localhost:9999'
            }
        }
        stage('Seeker Test Execute') {
            steps {
                // note: this is actually checking out within our java-sec-code code. but that's fine.
                sh 'git clone https://github.com/swright-synopsys/java-sec-code-test.git || true'
                sh 'python3 -m pip install webdriver_manager --user'
                sh 'cd java-sec-code-test; python3 install_firefox_driver.py'
                sh 'cd java-sec-code-test; python3 java-sec-code-test.py --host localhost --port 9999 --headless'
            }
        }
        stage('bbdba') {
            steps {
                withCredentials([string(credentialsId: 'BDBAAPI', variable: 'token')]) {
                    sh '''
                      curl -H \"Authorization: Bearer $token\" -T target/java-sec-code-1.0.0.jar https://protecode-sc.com/api/upload/
                    '''
                }
            }
        }
        stage('blackduck') {
               steps {
                synopsys_detect detectProperties: '-d --logging.level.com.synopsys.integration=DEBUG --detect.tools=DETECTOR', downloadStrategyOverride: [$class: 'ScriptOrJarDownloadStrategy']
               }
        }
        stage('oldpolaris') {
            steps {
                sh 'mkdir /tmp/polaris'
                sh 'echo $POLARIS_HOME'
                polaris arguments: 'analyze -w', polarisCli: 'sipse'
            }
        }
        stage('codedx') {
            steps {
                withEnv(["AWS=${AWS_HOST}", "PUBLIC=${PUBLIC_HOST}"]) {
                    withCredentials([string(credentialsId: 'CODEDXAPI', variable: 'token')]) {
                        sh '''
                          echo "$AWS"
                          echo "$PUBLIC"
                          # works like this: DATA='string'$VARSUBST'restofstring'

                          # Update coverity connect IP here
                          DATA='{\"server_url\":\"http://'$AWS':8000\",\"username\":\"admin\",\"ingest_all_components\":true,\"minimum_severity\":\"Info\",\"auto-refresh-interval\":false,\"available-during-analysis\":true,\"password\":{\"remembered\":true},\"selected_project\":\"java-sec-code\",\"selected_stream\":{\"value\":\"java-sec-code\",\"syncWith\":null}}'
                          curl -X \'PUT\' -k -H \'accept: application/json\' -H \'Content-Type: application/json\' -H "API-Key: $token" http://$AWS:8080/codedx/x/tool-connector-config/values/9 --data-raw $DATA

                          # update the Hub connector IP
                          DATA='{\"server_url\":\"https://'$PUBLIC'\",\"auth_type\":\"api_token\",\"security_risks\":true,\"license_risks\":true,\"operational_risks\":false,\"minimum_severity\":\"info\",\"matched_files\":true,\"upgrade_guidance\":true,\"bom_custom_fields\":false,\"comp_custom_fields\":false,\"comp_ver_custom_fields\":false,\"auto-refresh-interval\":false,\"available-during-analysis\":true,\"api_key\":{\"remembered\":true},\"project\":\"b9367623-6340-40c9-9422-4115f184b29c\",\"version\":\"831498c4-8c1a-42bb-914e-cc30af58bd51\"}'
                          curl -X \'PUT\' -k -H \'accept: application/json\' -H \'Content-Type: application/json\' -H "API-Key: $token" http://$AWS:8080/codedx/x/tool-connector-config/values/5 --data-raw $DATA

                          # update Seeker connector IP
                          DATA='{"host_url":"http://'$AWS':8088","auto-refresh-interval":false,"available-during-analysis":true,"access_token":{"remembered":true},"selected_project":"java-sec-code"}'
                          curl -X \'PUT\' -k -H \'accept: application/json\' -H \'Content-Type: application/json\' -H "API-Key: $token" http://$AWS:8080/codedx/x/tool-connector-config/values/6 --data-raw $DATA

                          # push results to codedx (java-sec-code project key is 1)
                          curl -X \'POST\' -k -H \'accept: application/json\' -H \'Content-Type: application/json\' -H "API-Key: $token" http://$AWS:8080/codedx/api/projects/1/analysis
                        '''
                    }
                }
            }
        }
    }
}
