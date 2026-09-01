// Staged container deployment: build, push, then roll out to DEV -> STAGE -> PROD
// with an approval gate between environments.
//
// Deployment runs through an Ansible Tower job template. The transport lives in
// deployTo() alone, so swapping it (see Jenkinsfile.ssh) touches nothing else.

def credentialForEnvironment( String environmentName )
{
    def credentials = [ DEV:   'deploy-machine-cred',
                        STAGE: 'deploy-machine-cred',
                        PROD:  'deploy-machine-cred' ]

    def credential = credentials[environmentName]
    if (!credential) error "No credential mapped for environment '${environmentName}'"
    return credential
}

def deployTo( String environmentName, String hostsCsv )
{
    def hosts = hostsCsv.split(',').collect { it.trim() }
    for (host in hosts)
    {
        echo "Deploying ${env.DEPLOY_REF} on ${host} (${environmentName})"

        ansibleTower credential: credentialForEnvironment(environmentName),
            inventory: 'app_hosts',
            jobTemplate: 'jt_app_deploy',
            jobType: 'run',
            limit: host,
            extraVars: """
                deploy_ref: ${env.DEPLOY_REF}
                latest_ref: ${env.IMAGE_PREFIX}latest
                build_url: ${env.BUILD_URL}
            """,
            throwExceptionWhenFail: true,
            importTowerLogs: true,
            towerLogLevel: 'full',
            towerServer: 'AnsibleTower',
            verbose: true
    }
}

pipeline
{
    agent
    {
        label 'docker-builder'
    }

    environment
    {
        // Assumes every environment can pull from this registry. Where zones have
        // separate registries, promote the image between them and make this
        // per-environment - see "One registry, or several" in the README.
        REGISTRY     = 'registry.example.com'
        NAMESPACE    = 'example'
        IMAGE        = 'app'
        IMAGE_PREFIX = "${env.REGISTRY}/${env.NAMESPACE}/${env.IMAGE}:"
    }

    parameters
    {
        choice( name: 'MODE',
                choices: ['Build only', 'Build and deploy', 'Deploy only'],
                description: 'Build only = build and push a new image; this is what automatic triggers use. Build and deploy = build, push, then roll out. Deploy only = deploy image_ref below without building.' )
        string( name: 'image_ref',
                defaultValue: 'latest',
                description: 'Deploy only: which existing image to deploy - build number, short commit hash, latest, or a sha256:... digest' )
        string( name: 'dev_hosts',
                defaultValue: 'na',
                description: 'Comma separated DEV hosts (na = skip this environment)' )
        string( name: 'stage_hosts',
                defaultValue: 'na',
                description: 'Comma separated STAGE hosts (na = skip this environment)' )
        string( name: 'prod_hosts',
                defaultValue: 'na',
                description: 'Comma separated PROD hosts (na = skip this environment)' )
    }

    stages
    {
        stage( 'Build' )
        {
            when { expression { params.MODE != 'Deploy only' } }
            steps
            {
                script
                {
                    def shortCommit = env.GIT_COMMIT.take(8)

                    docker.build(
                        "${env.IMAGE_PREFIX}${env.BUILD_ID}",
                        "--force-rm=true --pull " +
                        "--tag=${env.IMAGE_PREFIX}latest " +
                        "--tag=${env.IMAGE_PREFIX}${shortCommit} " +
                        "--label org.opencontainers.image.revision=${env.GIT_COMMIT} ."
                      )
                }
            }
        }

        stage( 'Push' )
        {
            when { expression { params.MODE != 'Deploy only' } }
            steps
            {
                script
                {
                    def shortCommit = env.GIT_COMMIT.take(8)

                    sh "docker push ${env.IMAGE_PREFIX}${env.BUILD_ID}"
                    sh "docker push ${env.IMAGE_PREFIX}latest"
                    sh "docker push ${env.IMAGE_PREFIX}${shortCommit}"
                }
            }
        }

        stage( 'Resolve image reference' )
        {
            when { expression { params.MODE != 'Build only' } }
            steps
            {
                script
                {
                    if ( params.MODE == 'Deploy only' )
                    {
                        // A digest is addressed with '@', a tag with ':'.
                        env.DEPLOY_REF = params.image_ref.startsWith('sha256:')
                            ? "${env.REGISTRY}/${env.NAMESPACE}/${env.IMAGE}@${params.image_ref}"
                            : "${env.IMAGE_PREFIX}${params.image_ref}"
                    }
                    else
                    {
                        env.DEPLOY_REF = "${env.IMAGE_PREFIX}${env.BUILD_ID}"
                    }

                    echo "Deploy target: ${env.DEPLOY_REF}"
                }
            }
        }

        stage( 'Deploy to DEV' )
        {
            when
            {
                allOf
                {
                    expression { params.MODE != 'Build only' }
                    expression { params.dev_hosts != 'na' }
                }
            }
            steps
            {
                script { deployTo('DEV', params.dev_hosts) }
            }
        }

        stage( 'Deploy to STAGE' )
        {
            when
            {
                allOf
                {
                    expression { params.MODE != 'Build only' }
                    expression { params.stage_hosts != 'na' }
                }
            }
            // Gate as a stage directive, not a step: the pipeline waits before
            // claiming an executor, so an unanswered prompt does not hold an agent.
            options { timeout(time: 4, unit: 'HOURS') }
            input
            {
                message "DEV deployed. Continue to STAGE?"
                ok "Deploy STAGE"
            }
            steps
            {
                script { deployTo('STAGE', params.stage_hosts) }
            }
        }

        stage( 'Deploy to PROD' )
        {
            when
            {
                allOf
                {
                    expression { params.MODE != 'Build only' }
                    expression { params.prod_hosts != 'na' }
                }
            }
            options { timeout(time: 4, unit: 'HOURS') }
            input
            {
                message "STAGE deployed. Continue to PROD?"
                ok "Deploy PROD"
            }
            steps
            {
                script { deployTo('PROD', params.prod_hosts) }
            }
        }
    }
}
