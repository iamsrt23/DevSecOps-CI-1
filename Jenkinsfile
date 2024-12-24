pipeline{
    agent {label 'build'}
    environment {
        registry = "iamsrt23/democicd"
        registryCredential = "dockerhub"
    }
    parameters {
        password(name: 'PASSWD', defaultValue: '', description: 'Please Enter you GitHub Password')
    }
    stages{
        stage('Checkout'){
            steps{

            }
        }
        stage('Stage1: Build'){
            steps{
                echo "Building Jar Component ..."
                sh "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn clean package"
                
            }
        }
        stage('Stage2: Code Coverage'){
            steps{
                echo "Running Code Coverage ...."
                sh  "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn jacoco:report "
                
            }
        }
        stage('Stage3 : SCA'){
            steps{
                echo "Running Software Composition Analysis"
                sh  "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn org.owasp:dependency-check-maven:check"
                
            }
        }
        stage('Stage4: SAST'){
            steps{
                echo "Running Static Applicatio security testing using SonarQube Scanner...."
                withSonarQubeEnv('mysonarqube'){
                    sh "mvn sonar:sonar -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check-report.json -Dsonar.depencencyCheck.htmlReportPath=target/dependency-check-report.html -Dsonar.projectName='cicd-demo'"
                }
                

            }
        }
        stage('Stage 5: QualityGates'){
            steps{
                echo "Running Quality Gates to verify the code quality"
                script{
                    timeout(time: 1, unit: 'MINUTES'){
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK'){

                            error 'Pipeline aborted due to quality gate failure: ${qg.status}'

                        }
                    }
                }
                
            }
        }
        stage('Stage6: BuildImage'){
            steps{
                echo "Build Docker Image"

                script{
                    docker.withRegistry('',registryCredential){
                        myImage = docker.build registry + ":$BUILD_NUMBER"
                        myImage.push()
                    }
                }


                
            }
        }
        stage('Stage 7: Scan Image'){
            steps{
                echo "Scanning Image for Vulnerabilities"
                sh "trivy image --scanners vuln --offline-scan iamsrt23/democicd:$BUILD_NUMBER > trivyresults.txt"

                
            }
        }
        stage('Stage 8: SmokeTest'){
            steps{
                echo "Smoke test the Image"
                sh "docker run -d --name smokerun -p 8000:8000 iamsrt23/democicd:$BUILD_NUMBER"
                sh "sleep 90; ./check.sh"
                sh "docker rm --force smokerun"
            }
        stage('Stage 9:Trigger Deployment'){
            steps{
                script{
                    TAG = "$BUILD_NUMBER"
                    echo "Trigger CD Pipeline"
                    build wait: false, job: 'demo-cd', parameters: [password(name: 'PASSWD', description: 'Please Enter you GitHub Password', value: params.PASSWD), string(name: 'IMAGETAG', value: TAG)]
                }
            }
        }
        }
    }
}