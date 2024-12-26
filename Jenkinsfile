node {
    def mavenTool = tool name: 'maven', type: 'maven'
    def jdkTool = tool name: 'jdk-21', type: 'jdk'
    def dockerImage = 'liloid/java-maven-app:latest'
    def ec2Host = '13.229.208.132'
    def ec2User = 'ec2-user'

    try {

        stage('Checkout') {
            checkout([$class: 'GitSCM', 
                branches: [[name: '*/dev']],
                userRemoteConfigs: [[url: 'file:///home/Documents/devops/cicd/simple-java-maven-app']]])
        }

        // stage('Debug Tests') {
        //     withEnv(["JAVA_HOME=${jdkTool}", "PATH+MAVEN=${mavenTool}/bin"]) { 
        //         sh 'pwd'
        //     }
        // }

        stage('Build') {
            withEnv(["JAVA_HOME=${jdkTool}", "PATH+MAVEN=${mavenTool}/bin"]) { 
                sh 'ls -la'
                sh 'mvn clean package'
                sh "docker build -t ${dockerImage} ."
            }
        }

        stage('Test') {
            withEnv(["JAVA_HOME=${jdkTool}", "PATH+MAVEN=${mavenTool}/bin"]) {
                try {
                    sh 'mvn test'
                } finally {
                    sh 'ls -la target/surefire-reports/'
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Manual Approval') {
            withEnv(["JAVA_HOME=${jdkTool}", "PATH+MAVEN=${mavenTool}/bin"]) {
                sh './jenkins/scripts/deliver.sh' 
                input message: 'Lanjutkan ke tahap Deploy?'
            }
        }

        stage('Deploy') {
            withEnv(["JAVA_HOME=${jdkTool}", "PATH+MAVEN=${mavenTool}/bin"]) {
                withDockerRegistry([credentialsId: 'docker-hub-credentials', url: '']) {
                    // Push Docker image to the registry
                    sh "docker push ${dockerImage}"
                }

                // Use SSH agent for connecting to EC2
                sshagent(['jenkins-docker-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@ec2-13-229-208-132.ap-southeast-1.compute.amazonaws.com <<'EOT'
                            set -e
                            sudo docker pull ${dockerImage}
                            sudo docker stop java-maven-app || true
                            sudo docker rm java-maven-app || true
                            sudo docker run -d --name java-maven-app -p 8080:8080 ${dockerImage}
                EOT
                    """
                }

                echo 'Application deployed successfully!'
                echo 'Aplikasi akan berjalan selama 1 menit...'
                sleep(time: 1, unit: 'MINUTES')
                echo 'Waktu habis, aplikasi akan dihentikan.'
            }
        }

    } catch (e) {
        echo "Build failed in stage '${currentBuild.currentResult}': ${e.getMessage()}"
        currentBuild.result = 'UNSTABLE' 
    } finally {
        if (currentBuild.result == 'UNSTABLE') {
            echo "Pipeline marked as unstable." 
        }
    }
}
