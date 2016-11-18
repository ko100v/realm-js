#!groovy
import groovy.json.JsonOutput

repoName = 'realm-js' // This is a global variable

def getSourceArchive() {
  checkout scm
  sh 'git clean -ffdx -e .????????'
  sshagent(['realm-ci-ssh']) {
    sh 'git submodule update --init --recursive'
  }
  
/*  checkout([
      $class: 'GitSCM',
      browser: [$class: 'GithubWeb', repoUrl: "git@github.com:realm/${repoName}.git"],
      extensions: [
          [$class: 'CleanCheckout'],
          [$class: 'WipeWorkspace'],
          [$class: 'SubmoduleOption', recursiveSubmodules: true]
      ],
      gitTool: 'native git',
      userRemoteConfigs: [[
          credentialsId: 'realm-ci-ssh',
          name: 'origin',
          refspec: '+refs/tags/*:refs/remotes/origin/tags/* +refs/heads/*:refs/remotes/origin/*',
          url: "git@github.com:realm/${repoName}.git"
      ]]
  ])
*/
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

def getNodeSpec(target) {
  if (target == "react-tests-android") {
    return "FastLinux"
  } else if (target == "node-linux") {
    return 'docker'
  }
  return 'osx_vegas'
}

def doDockerBuild(target) {
  return {
    timeout(25) { // 25 minutes
      node('docker') {
        getSourceArchive()
        sh "bash scripts/docker-test.sh ${target}"
      }
    }
  }
}

def doBuild(nodeSpec, target) {
  return {
    timeout(25) { // 25 minutes
      node(nodeSpec) {
        getSourceArchive()
        sh "bash scripts/test.sh ${target}"
      }
    }
  }
}

/*def doBuild(target, configuration) {
  node(getNodeSpec(target)) {
    getSourceArchive()
    timeout(25) { // 25 minutes
      if(target == 'node-linux') {
        sh "bash scripts/docker-test.sh node ${configuration}"
      } else {
        sh "bash scripts/test.sh ${target} ${configuration}"
      }
    }
    //step([$class: 'GitHubCommitStatusSetter', contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: "${target}_${configuration}"]])
  }
}
*/
stage('build') {
  parallel(
    eslint: doDockerBuild('eslint-ci'),
    jsdoc: doDockerBuild('jsdoc'),
    linux_node_debug: doDockerBuild('node Debug'),
    linux_node_release: doDockerBuild('node Release'),
    macos_node_debug: doBuild('osx_vegas', 'node Debug'),
    macos_node_release: doBuild('osx_vegas', 'node Release'),
    macos_realmjs_debug: doBuild('osx_vegas', 'realmjs Debug'),
    macos_react_tests_debug: doBuild('osx_vegas', 'react-tests Debug')
  )

/*  def configurations = ['Debug', 'Release']
  def targets = [
    'node',
    'node-linux',
    'realmjs',
    'react-tests',
    'react-example',
    'object-store',
    'test-runners'
  ]

  def jobs = [:]
  jobs['eslint'] = doBuild('eslint', 'Release')
  jobs['jsdoc'] = doBuild('jsdoc', 'Release')
  jobs['react-tests-android'] = doBuild('react-tests-android', 'Release')

  for (int i = 0; i < targets.size(); i++) {
    def targetName = targets[i];
    for (int j = 0; j < configurations.size(); j++) {
      def configurationName = configurations[j];

      jobs["${targetName}_${configurationName}"] = {
        doBuild(targetName, configurationName)
      }
    }
  }
*/
}