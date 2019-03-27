timestamps {
    node {
        stage('Checkout') {
            checkout scm
        }

        stage('Build') {
            sh "./gradlew --no-daemon clean build"
        }
    }
}