properties(
    [
        parameters(
            [
                string(defaultValue: '', name: 'RT_SERVER'),
                string(defaultValue: '', name: 'RT_RESOLVER_REPO'),
                string(defaultValue: '', name: 'RT_DEPLOYER_REPO'),
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

            def server = Artifactory.server params.RT_SERVER
            def rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
            def buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: 'clean artifactoryPublish'
            server.publishBuildInfo buildInfo
        }
    }
}