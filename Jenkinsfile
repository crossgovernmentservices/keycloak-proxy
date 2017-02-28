#!groovy

import groovy.json.JsonSlurper


node('golang') {

    properties([
        parameters([
            string(name: 'DISCOVERY_URL', defaultValue: 'https://gateway.civilservice.digital/auth/realms/ags'),
            string(name: 'PAAS_ORG', defaultValue: 'csd-sso'),
            string(name: 'PAAS_SPACE', defaultValue: 'sandbox'),
            string(name: 'GATEWAY_BRANCH', defaultValue: ''),
        ])
    ])

    stage("Check Discovery URL Available") {
        def url = "${DISCOVERY_URL}"
        try {
            response = httpRequest(
                httpMode: 'GET',
                url: url
            )
        } 
        catch (err) { 
            throw err
        }
    }

    stage("Source") {
        checkout scm
    }

    stage("Build") {
        withEnv(["PATH+GOROOT=/usr/local/go/bin", 
                "PATH+GOPATH=/home/ubuntu/work/bin",
                "GOPATH=/home/ubuntu/work"]) {

                ansiColor('xterm') {
                    sh "go version"
                    sh "make"
                }
        }
    }

    // Go tests not working so skipping stage
    // stage("Test") {
    //     withEnv(["PATH+GOROOT=/usr/local/go/bin", 
    //             "PATH+GOPATH=/home/ubuntu/work/bin",
    //             "GOPATH=/home/ubuntu/work"]) {
    //                 ansiColor('xterm') {
    //                     // sh "make test"
    //                 }
    //             }
    // }

    if (!BRANCH_NAME.startsWith('PR-')) {
        stage("Deploy") {
            def appName = cfAppName("kc-proxy")
            def url = "https://${appName}.cloudapps.digital"

            stash(
                name: "app",
                includes: ".cfignore,Procfile,bin/**,deploy-to-paas,update-proxy,*.yml,*.txt,*.pem,*.go,*.md,AUTHORS,LICENSE,templates/**,vendor/**,Godeps/**"
            )

            node('master') {

                unstash "app"

                def config = registerOIDCClient(appName)

                withEnv(config) {
                    echo "Registered OIDC client - ${PROXY_CLIENT_ID}"
                    retry(2) {
                        deployToPaaS(appName)
                    }
                }

                if (GATEWAY_BRANCH) {
                    // Assumes that default test client - Sue  My Brother/kc-proxy is already built
                    if (GATEWAY_BRANCH != 'master') {
                        // Only update Nginx proxy config if on a feature branch
                        updateProxy(appName)

                        // if (BRANCH_NAME != 'ags') {
                        //     slackSend color: success, message: "Deployed ${appName} of Keycloak Proxy to ${url}"
                        // }
                    }
                }

                echo "Deployed ${appName} to ${url}"
            }
        }
    }
}


def cfAppName(appName) {
    def branch = "${BRANCH_NAME.replace('_', '-')}"

    if (GATEWAY_BRANCH) {
        if (GATEWAY_BRANCH != 'master') {
                appName = "${appName}-${branch}-${GATEWAY_BRANCH}"
        }
    }

    return appName
}


def deployToPaaS(appName) {
    def smb_appName = 'sue-my-brother-kc-proxy'
    try {
        if (GATEWAY_BRANCH) {
            if (GATEWAY_BRANCH != 'master') {
                smb_appName = "${smb_appName}-g-${GATEWAY_BRANCH}"
            }
        }
    } catch (err) {
        // not a gateway dependent deploy, so do nothing
    }

    withEnv([
        "ORG=${PAAS_ORG}",
        "SPACE=${PAAS_SPACE}",
        "CF_APPNAME=${appName}",
        "SMB_NAME=${smb_appName}"]) {
        withCredentials([
            usernamePassword(
                credentialsId: 'paas-deploy',
                usernameVariable: 'CF_USER',
                passwordVariable: 'CF_PASSWORD')]) {

                ansiColor('xterm') {
                    sh "./deploy-to-paas"
                }
        }
    }
}

def updateProxy(appName) {
    withEnv([
        "CF_APPNAME=${appName}"]) {
        withCredentials([
                string(credentialsId: 'NGINX', variable: 'SERVER'), 
                file(credentialsId: 'ags-jenkins-slave.pem', variable: 'PEM_FILE')]) {
            ansiColor('xterm') {
                sh "./update-proxy"
            }
        }
    }
}
                
@NonCPS
def parseOIDCCreds(def json) {
    def config = new groovy.json.JsonSlurper().parseText(json)
    [
        "PROXY_CLIENT_ID=${config['client_id']}",
        "PROXY_CLIENT_SECRET=${config['client_secret']}",
    ]
}

def registerOIDCClient(appName) {
    withCredentials([
        string(credentialsId: 'KC_TOKEN', variable: 'TOKEN')]) {
            def url = "${DISCOVERY_URL}/clients-registrations/openid-connect"
            def json = "{\"redirect_uris\": [\"https://${appName}.cloudapps.digital/oauth/callback\"]}"
            def token = "bearer ${TOKEN}"

            def response = null
            timeout(5) {
                waitUntil {
                    try {
                        response = httpRequest(
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            requestBody: json,
                            url: url,
                            customHeaders: [[name: 'Authorization', value: token]]
                        )
                        return true
                    } catch (err) {
                        sleep(time: 30, unit: 'SECONDS')
                    }
                    return false
                }
            }

            parseOIDCCreds(response.content)
        }
}