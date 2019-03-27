properties(
    [
        parameters(
            [
                string(description: 'Artifactory server ID', defaultValue: 'artifactory-entplus-us-west', name: 'RT_SERVER_ID'),
                string(description: 'resolver repo name', defaultValue: 'jcenter', name: 'RT_RESOLVER_REPO'),
                string(description: 'deployer repo name', defaultValue: 'gradle-dev', name: 'RT_DEPLOYER_REPO'),
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
            if (! params.RT_SERVER_ID || ! params.RT_RESOLVER_REPO || ! params.RT_DEPLOYER_REPO) {
                throw new Exception("Missing required parameter")
            }

            def server = Artifactory.server params.RT_SERVER_ID
            def rtGradle = Artifactory.newGradleBuild()
            rtGradle.resolver server: server, repo: params.RT_RESOLVER_REPO
            rtGradle.deployer server: server, repo: params.RT_DEPLOYER_REPO
            rtGradle.useWrapper = true
            def buildInfo = rtGradle.run buildFile: 'build.gradle', tasks: 'clean artifactoryPublish'
            buildInfo.env.capture = true
            server.publishBuildInfo buildInfo
        }
    }
}