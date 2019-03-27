properties(
    [
        parameters(
            [
                string(defaultValue: '', name: 'RT_REPO_URL')
            ]
        )

    ]
)

timestamps {
    node {
        stage('Checkout') {
            checkout scm
        }

        stage('Build') {
            if (! params.RT_REPO_URL) {
                throw new Exception("Missing required parameter RT_REPO_URL")
            }
            sh "./gradlew --no-daemon clean build -PrtRepoUrl=${params.RT_REPO_URL}"
        }
    }
}