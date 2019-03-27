timestamps {
    node('generic') {
        stage('Checkout') {
            checkout scm
        }

        stage('Build') {
            sh "./gradlew build"
        }
    }
}