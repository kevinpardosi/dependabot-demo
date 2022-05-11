def MigrateWithVPN(){
    docker.image('wagely/node-openvpn:v0.0.6-bullseye').inside(){
        sh 'mkdir -p ~/.ssh'
        sh 'touch ~/.ssh/known_hosts'
        sh 'ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts'
        sh 'eval `ssh-agent -s` && ssh-add $JENKINS_PRIVATE_KEY && npm i'
    }
    sh '''
        cat $CONFIG > /tmp/config.ovpn
        echo "Public IP is now $(curl checkip.amazonaws.com)"
        phone_home="$(netstat -an | grep ':22 .*ESTABLISHED' | head -n1 | awk '{ split($5, a, ":"); print a[1] }')"
        echo $phone_home
        echo -e "\nroute $phone_home 255.255.255.255 net_gateway" >> /tmp/config.ovpn
        echo "route 169.254.0.0 255.255.0.0 net_gateway" >> /tmp/config.ovpn
        openvpn3 session-start --config /tmp/config.ovpn
        echo "Public IP is now $(curl checkip.amazonaws.com)"
        '''
    docker.image('wagely/node-openvpn:v0.0.6-bullseye').inside(){
        sh '''
        dig $DATABASE_HOST +time=10 +tries=3
        echo "MIGRATE WITH VPN"
        DB_HOST=$DATABASE_HOST DB_NAME=$DATABASE_NAME DB_USERNAME=$DATABASE_USR DB_PASSWORD=$DATABASE_PSW npm run migrate
        '''
    }
    sh '''
        openvpn3 session-manage --config /tmp/config.ovpn --disconnect
        echo "Public IP is now $(curl checkip.amazonaws.com)"
    '''
}
def MigrateWithoutVPN(){
    docker.image('wagely/node-openvpn:v0.0.6-bullseye').inside(){
        sh 'mkdir -p ~/.ssh'
        sh 'touch ~/.ssh/known_hosts'
        sh 'ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts'
        sh 'eval `ssh-agent -s` && ssh-add $JENKINS_PRIVATE_KEY && npm i'
        sh 'echo MIGRATE WITHOUT VPN'
        sh 'DB_HOST=$DATABASE_HOST DB_NAME=$DATABASE_NAME DB_USERNAME=$DATABASE_USR DB_PASSWORD=$DATABASE_PSW npm run migrate'
    }
}
def getMigrationDBAccountSecretName(){
    if(env.BRANCH_NAME == 'production'){
        return 'db-creds-prod'
    }else{
        return 'db-creds-staging'
    }
}
def getMigrationDBHostSecretName(){
    if(env.BRANCH_NAME == 'production'){
        return 'db-host-prod'
    }else{
        return 'db-host-staging'
    }
}
def getMigrationDBNameSecretName(){
    if(env.BRANCH_NAME == 'production'){
        return 'db-name-prod'
    }else{
        return 'db-name-staging'
    }
}
def getMigrationVPNConfigSecretName(){
    if(env.BRANCH_NAME == 'production'){
        return 'vpn-prod'
    }else{
        return 'vpn-staging'
    }
}
def getMigrationNodeName(){
    if(env.BRANCH_NAME == 'production'){
        return 'jenkins-cloud-agent-single'
    }else{
        return 'jenkins-cloud-agent'
    }
}

def notifySlack(String buildStatus) {
    buildStatus = buildStatus ?: 'SUCCESS'

    def color
    def mention = ''

    if (buildStatus == 'SUCCESS') {
        color = 'good'
    } else {
        color = 'danger'
        mention = '<!here> '
    }

    def blocks = [
        [
            "type": "section",
            "text": [
            "type": "mrkdwn",
            "text": "${mention}${JOB_NAME} #${BUILD_ID} ${buildStatus}"
            ]
        ],
        [
            "type": "divider"
        ],
        [
            "type": "section",
            "text": [
                "type": "mrkdwn",
                "text": "Duration:\n${currentBuild.durationString}"
            ]
        ],
        [
            "type": "actions",
            "elements": [
                [
                    "type": "button",
                    "text": [
                        "type": "plain_text",
                        "text": "More Detail",
                        "emoji": true
                    ],
                    "url": "${BUILD_URL}",
                    "action_id": "click_detail"
                ]
            ]
        ]
    ]

    slackSend(color: color, blocks: blocks)
}

pipeline{
    agent none
    options{
        disableConcurrentBuilds(abortPrevious: true)
        timeout(time:20)
    }
    environment{
        DBACCOUNTSECRETNAME = getMigrationDBAccountSecretName()
        DBHOSTSECRETNAME = getMigrationDBHostSecretName()
        DBNAMESECRETNAME = getMigrationDBNameSecretName()
        VPNCONFIGSECRETNAME = getMigrationVPNConfigSecretName()
        MIGRATIONNODENAME = getMigrationNodeName()
    }
    stages{
        stage ('audit'){
            when{
                expression{
                    env.BRANCH_NAME != 'production'
                }
            }
            agent{
                node {
                    label 'jenkins-cloud-agent'
                }
            }
            environment{
                HOME = '.'
                JENKINS_PRIVATE_KEY = credentials('Jenkins-private-key')
            }
            steps{
                script{
                    docker.image('wagely/node-openvpn:v0.0.6-bullseye').inside(){
                        sh 'mkdir -p ~/.ssh'
                        sh 'touch ~/.ssh/known_hosts'
                        sh 'ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts'
                        sh 'eval `ssh-agent -s` && ssh-add $JENKINS_PRIVATE_KEY && npm i && npm i better-npm-audit'
                        sh 'npm run audit'
                    }
                }
            }
        }
        stage('test'){
            when{
                expression{
                    env.BRANCH_NAME != 'production'
                }
            }
            agent{
                node { 
                    label 'jenkins-cloud-agent'
                }
            }
            environment{
                HOME = '.'
                JENKINS_PRIVATE_KEY = credentials('Jenkins-private-key')
                GIT_REPO_NAME = env.GIT_URL.replaceFirst(/^.*\/([^\/]+?).git$/, '$1').toLowerCase()
            }
            steps{
                script{
                    docker.image('wagely/node-openvpn:v0.0.6-bullseye').inside(){
                        sh 'mkdir -p ~/.ssh'
                        sh 'touch ~/.ssh/known_hosts'
                        sh 'ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts'
                        sh 'eval `ssh-agent -s` && ssh-add $JENKINS_PRIVATE_KEY && npm i'
                        sh 'npm run coverage'
                        stash includes: "coverage/lcov.info", name: "${GIT_REPO_NAME}-${env.GIT_BRANCH.replace('/','-')}-${env.BUILD_ID}-coverage", allowEmpty: true
                    }
                }
            }
        }
        stage('test-build'){
            environment{
                GIT_REPO_NAME = env.GIT_URL.replaceFirst(/^.*\/([^\/]+?).git$/, '$1').toLowerCase()
                JENKINS_PRIVATE_KEY_BASE64 = credentials('Jenkins-private-key-base64')
            }
            when{
                expression{
                    env.BRANCH_NAME != 'dev' &&
                    env.BRANCH_NAME != 'production' &&
                    env.BRANCH_NAME != 'master' &&
                    env.BRANCH_NAME !=~ 'hotfix-*' &&
                    env.BRANCH_NAME !=~ 'release-*'
                }
            }
            agent{
                node{
                    label 'jenkins-cloud-agent'
                }
            }
            steps{
                sh '''
                ./.circleci/deploy-marker.sh
                docker build -t $GIT_REPO_NAME:$BUILD_ID-test --build-arg SSH_KEY="$JENKINS_PRIVATE_KEY_BASE64" -f .ci/Dockerfile .
                docker rmi $GIT_REPO_NAME:$BUILD_ID-test'''
            }
        }
        stage('analyze'){
            when{
                expression{
                    env.BRANCH_NAME != 'production'
                }
            }
            agent{
                node { 
                    label 'jenkins-cloud-agent'
                }
            }
            environment{
                HOME = '.'
                SONAR_HOST_URL = 'https://sonarcloud.io/'
                SONAR_TOKEN = credentials('sonarcloud-token')
                JENKINS_PRIVATE_KEY = credentials('Jenkins-private-key')
                GIT_REPO_NAME = env.GIT_URL.replaceFirst(/^.*\/([^\/]+?).git$/, '$1').toLowerCase()
            }
            steps{
                script{
                    def sonarParams = ""
                    if (env.CHANGE_ID) {
                        sonarParams = "-Dsonar.pullrequest.key=${env.CHANGE_ID} -Dsonar.projectVersion=${env.GIT_COMMIT}"
                    } else {
                        sonarParams = "-Dsonar.branch.name=${env.BRANCH_NAME} -Dsonar.projectVersion=${env.GIT_COMMIT}"
                    }
                    docker.image('wagely/node-openvpn:v0.0.6-bullseye').inside(){
                        sh 'mkdir -p ~/.ssh'
                        sh 'touch ~/.ssh/known_hosts'
                        sh 'ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts'
                        sh 'eval `ssh-agent -s` && ssh-add $JENKINS_PRIVATE_KEY && npm i'
                    }
                    docker.image('sonarsource/sonar-scanner-cli').inside(){
                        unstash name: "${GIT_REPO_NAME}-${env.GIT_BRANCH.replace('/','-')}-${env.BUILD_ID}-coverage"
                        sh "sonar-scanner ${sonarParams} -Dproject.settings=sonar-project.properties"
                    }
                }
            }
        }
        stage('migration'){
            environment{
                DATABASE = credentials("$DBACCOUNTSECRETNAME")
                DATABASE_HOST = credentials("$DBHOSTSECRETNAME")
                DATABASE_NAME = credentials("$DBNAMESECRETNAME")
                CONFIG = credentials("$VPNCONFIGSECRETNAME")
                JENKINS_PRIVATE_KEY = credentials('Jenkins-private-key')
            }
            when{
                expression{
                    env.BRANCH_NAME == 'master' || 
                    env.BRANCH_NAME == 'production' ||
                    env.BRANCH_NAME ==~ 'hotfix-*' ||
                    env.BRANCH_NAME ==~ 'release-*'
                }
            }
            agent{
                node { 
                    label "${MIGRATIONNODENAME}"
                }
            }
            steps{
                script{
                    if(env.BRANCH_NAME == 'production'){
                        MigrateWithVPN()
                    }else{
                        MigrateWithoutVPN()
                    }
                }
            }            
        }
        stage('build and push image'){
            agent{
                node{
                    label 'jenkins-cloud-agent'
                }                
            }
            environment{
                GIT_REPO_NAME = env.GIT_URL.replaceFirst(/^.*\/([^\/]+?).git$/, '$1').toLowerCase()
                ECR_CREDENTIAL = 'aws-push-image'
                JENKINS_PRIVATE_KEY_BASE64 = credentials('Jenkins-private-key-base64')
                GITHUB_CRED = credentials('server_key')
            }
            when{
                expression{
                    env.BRANCH_NAME == 'dev' ||
                    env.BRANCH_NAME == 'production' ||
                    env.BRANCH_NAME == 'master' ||
                    env.BRANCH_NAME ==~ 'hotfix-*' ||
                    env.BRANCH_NAME ==~ 'release-*'
                }
            }
            steps{
                script{
                    docker.withRegistry('https://' + env.ECR_URL,"ecr:${env.ECR_REGION}:${env.ECR_CREDENTIAL}"){
                        sh './.circleci/deploy-marker.sh'
                        def devBranch = ''
                        if(env.BRANCH_NAME == 'dev'){
                            devBranch = '-dev'
                        }
                        def builtImage = docker.build("${GIT_REPO_NAME}${devBranch}:${env.BUILD_ID}",'--build-arg SSH_KEY="$JENKINS_PRIVATE_KEY_BASE64" -f ./.ci/Dockerfile .')
                        if(env.BRANCH_NAME == 'production'){
                            //stable
                            builtImage.push('stable')
                        }else{
                            //latest
                            builtImage.push('latest')
                        }
                        builtImage.push("${env.GIT_BRANCH.replace('/','-')}-${env.BUILD_ID}")
                        builtImage.push("${env.GIT_COMMIT}")
                    }
                }
            }
            post {
                always {
                    jiraSendBuildInfo site: 'wagely.atlassian.net'
                }
            }
        }
    }
    
    post{
        success{
            node('jenkins-cloud-agent'){
                notifySlack("SUCCESS")
            }
        }
        failure{
            node('jenkins-cloud-agent'){
                notifySlack("FAILED")
            }
        }
        always {
            node('jenkins-cloud-agent'){
                cleanWs()
            }
        }
    }
}