pipeline {
	agent any

		stages {
			stage ('Display versions') {
				steps {
					sh '''
						docker -v
						python -V
						ansible --version
						molecule --version
						'''
				}
			}
			stage('Build') {
				steps {
					sh '''
						echo 'Building...jk, nothing to build yet'
						'''
				}
			}
			stage('Test') {
				steps {
					sh '''
						export LC_ALL=en_US.utf8
						export LANG=en_US.utf8
#					sudo /bin/molecule test
						'''
				}
			}
			stage('Deploy') {
				steps {
					sh '''
						echo 'Deploying....jk, nothing to deploy yet'
						'''
				}
			}
		}

  post {
    success {
        setBuildStatus("Build succeeded - happy face", "SUCCESS");
    }
    failure {
        setBuildStatus("Build failed - sad face", "FAILURE");
    }
  }
}
