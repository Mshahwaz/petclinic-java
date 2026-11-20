pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        // Fill these values when ready
        GIT_REPOSITORY = 'https://github.com/Mshahwaz/petclinic-java.git'
        GIT_BRANCH = 'main'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    if (!env.GIT_REPOSITORY?.trim() ||
                        !env.GIT_BRANCH?.trim()) {
                        error('Please configure GIT_REPOSITORY and GIT_BRANCH.')
                    }
                }

                git branch: "${GIT_BRANCH}",
                    url: "${GIT_REPOSITORY}"
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    docker.image('maven:3.9.9-eclipse-temurin-17').inside('-v /var/lib/jenkins/mvn-rep:/mvn-rep'){
                        sh '''
								set -eux
						        echo "=== Maven local repository ==="
						        mvn help:evaluate \
						            -Dexpression=settings.localRepository \
									-Dmaven.repo.local=/mvn-rep \
						            -q -DforceStdout
						
						        echo "=== Build and Test ==="
						        mvn -B -Dmaven.repo.local=/mvn-rep clean verify 
						'''
                    }
                }
            }
        }
		
		stage('SAST - SonarQube') {
		    steps {
		        withCredentials([
		            string(
		                credentialsId: 'sonarqube-token',
		                variable: 'SONAR_TOKEN'
		            )
		        ]) {
		            script {
		                docker.image(
		                    'maven:3.9.9-eclipse-temurin-21'
		                ).inside(
		                    '--network host -v /var/lib/jenkins/mvn-rep:/mvn-rep'
		                ) {
		                    sh '''
		                        set -eu
		
		                        echo "=== SonarQube Analysis ==="
		
		                        mvn -B \
		                          -Dmaven.repo.local=/mvn-rep \
		                          org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar \
		                          -Dsonar.projectKey=petclinic \
		                          -Dsonar.host.url=http://127.0.0.1:9000 \
		                          -Dsonar.java.source=17
		                    '''
		                }
		            }
		        }
		    }
		}

    }

    post {
        always {
            junit(
                testResults: 'target/surefire-reports/*.xml',
                allowEmptyResults: true
            )
			archiveArtifacts artifacts: 'target/*.jar, target/site/**', allowEmptyArchive: true

        }

        success {
            echo 'PetClinic build and tests completed successfully.'
        }

        failure {
            echo 'PetClinic pipeline failed. Check the console output.'
        }
    }
}
