properties(
    [
        parameters(
            [
                string(defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
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
            if (! params.RT_SERVER_ID || params.RT_RESOLVER_REPO || params.RT_DEPLOYER_REPO) {
                throw new Exception("Missing required parameter RT_REPO_URL")
            }

            def server = Artifactory.server params.RT_SERVER_ID
            def rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
            def buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: 'clean artifactoryPublish'
            server.publishBuildInfo buildInfo
        }
    }
}