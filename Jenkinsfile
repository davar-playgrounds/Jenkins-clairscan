pipeline {
  agent any
  environment {
        hostssh='ssh -l root 172.17.0.1'
  }
  parameters{
    string(name: 'targetImage', defaultValue: 'centos:7', description: 'Please enter the image you wish to scan' )
  }
  stages {        
    stage('Prep Environment') {
      steps {
        sh '''
echo "starting scan services"

$hostssh docker network create scanning 
$hostssh docker run -p 5432:5432 -d --net=scanning --name db arminc/clair-db:latest
echo "Snoozing for DB to start" && sleep 30
$hostssh docker run -p 6060:6060 -d --net=scanning --link db:postgres --name clair arminc/clair-local-scan:v2.0.1

echo "Snooze and check they are up"
sleep 8
$hostssh docker ps
'''
      }
    }
    stage('Run Scanner') {
      steps {
        sh '''
$hostssh docker pull $targetImage
echo "Run a scan"
$hostssh mkdir /tmp/$BUILD_TAG || echo "Folder was already present"
$hostssh chmod 777 /tmp/$BUILD_TAG
$hostssh docker run --net=scanning --rm --name=scanner --link=clair:clair -v "/tmp/$BUILD_TAG/:/var/log/:z" -v \'/var/run/docker.sock:/var/run/docker.sock:z\' objectiflibre/clair-scanner --clair="http://clair:6060" --ip="scanner" -r "var/log/$BUILD_NUMBER.json" -t Medium $targetImage || echo "Warning Error returned in results remove this echo if you really want to stop the job here"
'''
      }
    }
    stage('Report Collection') {
      steps {
        sh '''
echo "We should gather the reports"
mkdir /tmp/$BUILD_TAG || echo "Folder was already present"
scp root@172.17.0.1:/tmp/$BUILD_TAG/$BUILD_NUMBER.json $WORKSPACE/
# /var/lib/jenkins/jobs/Jenkins-BlueOcean-Demo-Scan/branches/dev/workspace
#/tmp/$BUILD_TAG/
#pwd
#env
'''
archiveArtifacts '$BUILD_NUMBER.json'
      }
    }
  }
  post {
    always {
        echo 'It was inevitable'
      sh ''' 
        echo "Begin Cleanup"
        $hostssh docker container stop db || echo "."
        $hostssh docker container stop clair || echo "."
        $hostssh docker container rm db || echo "."
        $hostssh docker container rm clair || echo "."
        $hostssh docker network rm scanning || echo "."
        $hostssh docker container prune -f || echo "."
        $hostssh docker image prune -f || echo "."
        $hostssh rm -rf /tmp/$BUILD_TAG || echo "."
        rm -rf /tmp/$BUILD_TAG || echo "."
        echo "Cleanup finished"
      '''
        // deleteDir() /* clean up our workspace */
    }
    success {
        echo 'I succeeeded!'
    }
    unstable {
        echo 'I am unstable :/'
    }
    changed {
        echo 'Things were different before...'
    }
    failure {
        echo 'It did not end well'
    }
  }
}
