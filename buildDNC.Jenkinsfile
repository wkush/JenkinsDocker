  def dockerImage;

  node('docker') {
    stage('SCM') {
      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/wkush/JenkinsDocker']]]);
      }
      stage('build'){
        dockerImage = docker.build('garrumph/agent-dnc:v$BUILD_NUMBER','./dotnetcore');
      }
      stage('push'){
        docker.withRegistry('', 'dockerhubcredsgarrumph'){
          dockerImage.push();
        }
      }
    }

