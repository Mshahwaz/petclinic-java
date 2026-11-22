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
				script {
                    env.IMAGE_ID = "petclinic:${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
                }
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
		                    '--network host -v /var/lib/jenkins/mvn-rep:/mvn-rep -v /var/lib/jenkins/sonar-cache:/sonar-cache'
		                ) {
		                    sh '''
		                        set -eu
		                        echo "=== SonarQube Analysis ==="
		
		                        mvn -B \
		                          -Dmaven.repo.local=/mvn-rep \
		                          org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar \
		                          -Dsonar.projectKey=petclinic \
		                          -Dsonar.host.url=http://127.0.0.1:9000 \
		                          -Dsonar.java.source=17 \
		                          -Dsonar.userHome=/sonar-cache \
								  -Dsonar.token="$SONAR_TOKEN"
		                    '''
		                }
		            }
		        }
		    }
		}
		stage('SCA - Dependency Check') {
		    steps {
		        withCredentials([
		            string(
		                credentialsId: 'nvd-api-key',
		                variable: 'NVD_API_KEY'
		            )
		        ]) {
		            script {
		                docker.image(
		                    'maven:3.9.9-eclipse-temurin-17'
		                ).inside(
		                    '-v /var/lib/jenkins/mvn-rep:/mvn-rep -v /var/lib/jenkins/dependency-check-data:/dependency-check-data'
		                ) {
		                    sh '''
		                        set -eu
		
		                        echo "=== SCA - Dependency Check ==="
		
		                        mvn -B \
		                          -Dmaven.repo.local=/mvn-rep \
		                          -DnvdApiKeyEnvironmentVariable=NVD_API_KEY \
		                          -Ddata.directory=/dependency-check-data \
		                          org.owasp:dependency-check-maven:12.2.2:check
		                    '''
		                }
		            }
		        }
		    }
		}
		stage('Docker Image Build'){
			steps {
				sh '''
				set -eu

				echo "=========Validating Docker Build Inputs========"

				test -f Dockerfile
				test -f target/*.jar

				echo "=========Building Docker Image================="

				docker build -t "$IMAGE_ID" .

				echo "=========Verifying Docker Image================"

				docker image inspect "$IMAGE_ID" --format 'Image ID: {{.Id}}'

				docker images "$IMAGE_ID"
				'''
			}
		}
		stage('Trivy Image Scanning'){
			steps {
				sh '''
				set -eu
				docker run --rm \
				  -v /var/run/docker.sock:/var/run/docker.sock \
				  -v /var/lib/jenkins/trivy-cache:/root/.cache/ \
				  -v /var/lib/jenkins/trivy-reports:/reports \
				  aquasec/trivy:0.76.2 \
				  image \
				  --severity HIGH,CRITICAL \
				  --format json \
				  --output /reports/petclinic-report.json \
	              "$IMAGE_ID"
				'''
			}
		}
		stage('Push Image to ECR Public') {
	    environment {
	        ECR_PUBLIC_URI = 'public.ecr.aws/m3f2d4v8/petclinic'
	        AWS_REGION = 'us-east-1'
	    }
	
	    steps {
	        sh '''
	            set -eu
				
	            REMOTE_IMAGE="${ECR_PUBLIC_URI}:$(git rev-parse --short HEAD)"
	
	            echo "=== Verifying AWS Identity ==="
	            aws sts get-caller-identity
	
	            echo "=== Authenticating with ECR Public ==="
	            ECR_PASSWORD="$(aws ecr-public get-login-password \
	                --region "$AWS_REGION")"
	
	            printf '%s' "$ECR_PASSWORD" | docker login \
	                --username AWS \
	                --password-stdin public.ecr.aws
	
	            unset ECR_PASSWORD
	
	            echo "=== Tagging Docker Image ==="
	            docker tag "$IMAGE_TAG" "$REMOTE_IMAGE"
	
	            echo "=== Pushing Docker Image ==="
	            docker push "$REMOTE_IMAGE"
	
	            echo "=== Push Completed ==="
	            echo "$REMOTE_IMAGE"
	        '''
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
			publishHTML(target: [
	            reportDir: 'target',
	            reportFiles: 'dependency-check-report.html',
	            reportName: 'OWASP Dependency-Check Report',
	            reportTitles: 'Dependency-Check Report',
	            keepAll: true,
	            alwaysLinkToLastBuild: true,
	            allowMissing: false
        ])

        }

        success {
            echo 'PetClinic build and tests completed successfully.'
        }

        failure {
            echo 'PetClinic pipeline failed. Check the console output.'
        }
    }
}
