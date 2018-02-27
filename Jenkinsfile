node('centos7-docker-4c-2g') {
  withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'WORLD', usernameVariable: 'HELLO')]){
    echo "${env.HELLO} ${env.WORLD}"
    sh "docker login nexus3.edgexfoundry.org:10001 -u ${env.HELLO} -p ${env.WORLD}"
  }
  withDockerRegistry([credentialsId: "docker", url: "https://nexus3.edgexfoundry.org:10001"]){
    sh "docker pull nexus3.edgexfoundry.org:10001/edgexfoundry/docker-device-bacnet:0.2"
  }
}

