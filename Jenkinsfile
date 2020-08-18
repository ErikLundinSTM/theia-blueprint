/**
 * This Jenkinsfile builds Theia across the major OS platforms
 */

pipeline {
    agent none
    options {
        timeout(time: 3, unit: 'HOURS') 
    }
    stages {
        stage('Build') {
            parallel {
                stage('Create Linux Installer') {
                    agent {
                        kubernetes {
                            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: theia-dev
    image: thegecko/theia-dev
    command:
    - cat
    tty: true
    resources:
      limits:
        memory: "4Gi"
        cpu: "2"
      requests:
        memory: "4Gi"
        cpu: "2"
    volumeMounts:
    - name: yarn-cache
      mountPath: /.cache
    - name: electron-cache
      mountPath: /.electron-gyp
  volumes:
  - name: yarn-cache
    emptyDir: {}
  - name: electron-cache
    emptyDir: {}
"""
                        }
                    }
                    steps {
                        container('theia-dev') {
                            script {
                                buildInstaller()
                            }
                        }
                        stash name: 'linux'
                    }
                }
                stage('Create Mac Installer') {
                    agent {
                        label 'macos'
                    }
                    steps {
                        script {
                            buildInstaller()
                        }
                        stash name: 'mac'
                    }
                }
                stage('Create Windows Installer') {
                    agent {
                        label 'windows'
                    }
                    steps {
                        script {
                            buildInstaller()
                        }
                        stash name: 'win'
                    }
                }
            }
        }
        stage('Sign and Upload') {
            parallel {
                stage('Upload Linux') {
                    agent any
                    steps {
                        unstash 'linux'
                        script {
                            uploadInstaller('linux')
                        }
                    }
                }
                stage('Sign and Upload Mac') {
                    agent any
                    steps {
                        unstash 'mac'
                        script {
                            signInstaller('dmg', 'macsign')
                            notarizeInstaller()
                            uploadInstaller('macos')
                        }
                    }
                }
                stage('Sign and Upload Windows') {
                    agent any
                    steps {
                        unstash 'win'
                        script {
                            signInstaller('exe', 'winsign')
                            uploadInstaller('windows')
                        }
                    }
                }
            }
        }
    }
}

def buildInstaller() {
    checkout scm
    sh "printenv"
    sh "yarn cache clean"
    sh "yarn --frozen-lockfile --force"
    sh "yarn package"
}

def signInstaller(String ext, String url) {
    List installers = findFiles(glob: "dist/*.${ext}")
    if (installers.size() > 0) {
        sh "curl -o dist/signed-${installers[0].name} -F file=@${installers[0].path} http://build.eclipse.org:31338/${url}.php"
    }
}

def notarizeInstaller() {
    String service = 'http://172.30.206.146:8383/macos-notarization-service'
    List installers = findFiles(glob: "dist/signed-*.dmg")

    if (installers.size() > 0) {
        String response = sh(script: "$(curl -s -X POST -F file=@${installers[0].path} -F 'options={'primaryBundleId': 'app-bundle', 'staple': true};type=application/json' ${service}/notarize)", returnStdout: true)
        
        def jsonSlurper = new JsonSlurper()
        def json = jsonSlurper.parseText(response)
        String uuid = json.uuid

        while(json.status == 'IN_PROGRESS') {
            sleep(10000)
            response = new URL("${service}/${uuid}/status").text
            json = jsonSlurper.parseText(response)
        }

        if (json.status != 'COMPLETE') {
            error("Failed to notarize ${installers[0].name}")
        }

        sh "curl -o dist/notarized-${installers[0].name} ${service}/${uuid}/download"
    }
}

def uploadInstaller(String platform) {
    def packageJSON = readJSON file: "package.json"
    String version = "${packageJSON.version}"
    sshagent(['projects-storage.eclipse.org-bot-ssh']) {
        sh "ssh genie.theia@projects-storage.eclipse.org rm -rf /home/data/httpd/download.eclipse.org/theia/${version}/${platform}"
        sh "ssh genie.theia@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/theia/${version}/${platform}"
        sh "scp -r dist/*theia* genie.theia@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/theia/${version}/${platform}"
    }
}
