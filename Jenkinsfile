/**
 * This Jenkinsfile builds Theia across the major OS platforms
 */

pipeline {
    agent none
    environment {
        def packageJSON = readJSON file: "package.json"
        PACKAGE_VERSION = "${packageJSON.version}"
        BUILD_TIMEOUT = 180
    }
    stages {
        stage('Build') {
            parallel {
                stage('Create Linux Installer') {
                    agent {
                        kubernetes {
                            label 'node-pod'
                            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
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
                        timeout(time: "${env.BUILD_TIMEOUT}") {
                            container('node') {
                                script {
                                    buildInstaller()
                                }
                            }
                        }
                        stash includes: 'dist/theia*', name: 'linux'
                    }
                }
                stage('Create Mac Installer') {
                    agent {
                        label 'macos'
                    }
                    steps {
                        timeout(time: "${env.BUILD_TIMEOUT}") {
                            script {
                                buildInstaller()
                            }
                        }
                        stash includes: 'dist/theia*', name: 'mac'
                    }
                }
                stage('Create Windows Installer') {
                    agent {
                        label 'windows'
                    }
                    steps {
                        timeout(time: "${env.BUILD_TIMEOUT}") {
                            script {
                                buildInstaller()
                            }
                        }
                        stash includes: 'dist/theia*', name: 'win'
                    }
                }
            }
        }
        stage('Sign and Upload') {
            parallel {
                node {
                    stage('Upload Linux') {
                        steps {
                            unstash 'linux'
                            script {
                                uploadInstaller('linux')
                            }
                        }
                    }
                    stage('Sign and Upload Mac') {
                        steps {
                            unstash 'mac'
                            script {
                                signInstaller('dmg', 'macsign')
                                uploadInstaller('macos')
                            }
                        }
                    }
                    stage('Sign and Upload Windows') {
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
        String installer = installers[0]
        sh "curl -o dist/signed-${installer.name} -F file=@${installer.path} http://build.eclipse.org:31338/${url}.php"
    }
}

def uploadInstaller(String platform) {
    sshagent(['projects-storage.eclipse.org-bot-ssh']) {
        sh "ssh genie.theia@projects-storage.eclipse.org rm -rf /home/data/httpd/download.eclipse.org/theia/${env.PACKAGE_VERSION}/${platform}"
        sh "ssh genie.theia@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/theia/${env.PACKAGE_VERSION}/${platform}"
        sh "scp -r dist/* genie.theia@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/theia/${env.PACKAGE_VERSION}/${platform}"
    }
}
