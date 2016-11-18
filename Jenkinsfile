#!groovy
import groovy.json.JsonOutput

repoName = 'realm-js' // This is a global variable

def getSourceArchive() {
  checkout scm
  sh 'git clean -ffdx -e .????????'
  sshagent(['realm-ci-ssh']) {
    sh 'git submodule update --init --recursive'
  }
}

def readGitTag() {
  sh "git describe --exact-match --tags HEAD | tail -n 1 > tag.txt 2>&1 || true"
  def tag = readFile('tag.txt').trim()
  return tag
}

def readGitSha() {
  sh "git rev-parse HEAD | cut -b1-8 > sha.txt"
  def sha = readFile('sha.txt').readLines().last().trim()
  return sha
}

def getVersion(){
  def dependencies = readProperties file: 'dependencies.list'
  def gitTag = readGitTag()
  def gitSha = readGitSha()
  if (gitTag == "") {
    return "${dependencies.VERSION}-g${gitSha}"
  }
  else {
    return dependencies.VERSION
  }
}

def setBuildName(newBuildName) {
  currentBuild.displayName = "${currentBuild.displayName} - ${newBuildName}"
}

def gitTag
def gitSha
def dependencies
def version

stage('check') {
  node('docker') {
    getSourceArchive()

    dependencies = readProperties file: 'dependencies.list'

    gitTag = readGitTag()
    gitSha = readGitSha()
    version = getVersion()
    echo "tag: ${gitTag}"
    if (gitTag == "") {
      echo "No tag given for this build"
      setBuildName("${gitSha}")
    } else {
      if (gitTag != "v${dependencies.VERSION}") {
        echo "Git tag '${gitTag}' does not match v${dependencies.VERSION}"
      } else {
        echo "Building release: '${gitTag}'"
        setBuildName("Tag ${gitTag}")
      }
    }
    echo "version: ${version}"
  }
}

def doDockerBuild(target, postStep = null) {
  return {
    timeout(25) { // 25 minutes
      node('docker') {
        getSourceArchive()
        sh "bash scripts/docker-test.sh ${target}"
        if(postStep) {
          postStep.call()
        }
      }
    }
  }
}

def doBuild(nodeSpec, target, postStep = null) {
  return {
    timeout(25) { // 25 minutes
      node(nodeSpec) {
        getSourceArchive()
        sh "bash scripts/test.sh ${target}"
        if(postStep) {
          postStep.call()
        }
      }
    }
  }
}

stage('build') {
  parallel(
    eslint: doDockerBuild('eslint-ci', {
      step([$class: 'CheckStylePublisher', canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: 'eslint.xml', unHealthy: ''])
    }),
    jsdoc: doDockerBuild('jsdoc'),
    linux_node_debug: doDockerBuild('node Debug'),
    linux_node_release: doDockerBuild('node Release'),
    macos_node_debug: doBuild('osx_vegas', 'node Debug'),
    macos_node_release: doBuild('osx_vegas', 'node Release'),
    macos_realmjs_debug: doBuild('osx_vegas', 'realmjs Debug'),
    macos_realmjs_release: doBuild('osx_vegas', 'realmjs Release'),
    macos_react_tests_debug: doBuild('osx_vegas', 'react-tests Debug'),
    macos_react_tests_release: doBuild('osx_vegas', 'react-tests Release'),
    macos_react_example_debug: doBuild('osx_vegas', 'react-example Debug'),
    macos_react_example_release: doBuild('osx_vegas', 'react-example Release'),
    macos_object_store_debug: doBuild('osx_vegas', 'object-store Debug'),
    macos_object_store_release: doBuild('osx_vegas', 'object-store Release'),
    android_react_tests: doBuild('FastLinux', 'react-tests-android', {
      sh "cat tests/react-test-app/tests.xml"
      junit 'tests/react-test-app/tests.xml'
    })
  )

/*  def configurations = ['Debug', 'Release']
  def targets = [
    'test-runners'
  ]
*/
}