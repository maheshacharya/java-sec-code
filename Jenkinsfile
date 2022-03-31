pipeline {
    agent {
        docker {
            image 'gsasig/magic-tool:latest'
        }
    }

    environment {
        POLARIS_HOME = '/tmp/polaris'
    }

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
    }

    stages {
        stage('Build') {
            steps {
                sh "AWS_HOST = `curl -s http://169.254.169.254/latest/meta-data/local-hostname`"
                sh "echo My hostname is: ${AWS_HOST}"
                sh 'echo $AWS_HOST'
                sh "hostname"

                // Get some code from a GitHub repository
                git 'https://github.com/maheshacharya/java-sec-code.git'

                // Run Maven on a Unix agent.
                sh "mvn clean package -DskipTests"
    
            }
        }
        stage('Run Java Sec Code with Seeker Agent') {
            steps {
                // DL seeker agent
                sh "wget --no-check-certificate --content-disposition \'http://ip-172-31-32-101:8088/rest/api/latest/installers/agents/binaries/JAVA?agentVersion=2022.2.0&flavor=JAR\'"
                // instrumented w/seeker agent
                sh 'nohup java -javaagent:./seeker-agent.jar  -Dseeker.project.key=java-sec-code -Dseeker.server.url=http://ip-172-31-32-101:8088 -jar target/java-sec-code-1.0.0.jar --server.port=9999 > jsc.out 2>&1 &'
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
                withCredentials([string(credentialsId: 'BDBAAPI', variable: 'bdbaapi')]) {
                    sh "curl -H \'Authorization: Bearer $BDBAAPI\' -T target/java-sec-code-1.0.0.jar https://protecode-sc.com/api/upload/"
                }
            }
        }
        stage('blackduck') {
               steps {
                synopsys_detect detectProperties: '', downloadStrategyOverride: [$class: 'ScriptOrJarDownloadStrategy']
               }
        }
        stage('polaris') {
            steps {
                sh 'mkdir /tmp/polaris'
                sh 'printenv | grep POLARIS'
                sh 'echo $POLARIS_HOME'
                polaris arguments: 'analyze -w', polarisCli: 'sipse'
            }
        }
        stage('codedx') {
            steps {
                withCredentials([string(credentialsId: 'CODEDXAPI', variable: 'codedxapi')]) {
                    // code dx project key is 1
                    sh "curl -X \'POST\' -k -H \'accept: application/json\' -H \'Content-Type: application/json\' -H \'API-Key: $CODEDXAPI\' http://ip-172-31-32-101:8080/codedx/api/projects/1/analysis"
                }
            }
        }
        
    }
}
