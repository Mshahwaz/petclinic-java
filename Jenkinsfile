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
                    docker.image('maven:3.9.9-eclipse-temurin-17').inside {
                        sh '''
        						set -eux

				        		echo "=== Removing old Maven cache from workspace ==="
						        rm -rf -- './?/.m2'

						        echo "=== Maven local repository ==="
						        mvn -B -Dmaven.repo.local=/tmp/maven-repository \
						            help:evaluate \
						            -Dexpression=settings.localRepository \
						            -q -DforceStdout
						
						        echo
						        echo "=== Checking workspace for .m2 ==="
						        find . -maxdepth 4 -type d -name ".m2" -print
						
						        echo "=== Build and Test ==="
						        mvn -B -Dmaven.repo.local=/tmp/maven-repository clean verify 
						'''
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
        }

        success {
            echo 'PetClinic build and tests completed successfully.'
        }

        failure {
            echo 'PetClinic pipeline failed. Check the console output.'
        }
    }
}
