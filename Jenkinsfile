node('centos7-docker-4c-2g') {
  withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'WORLD', usernameVariable: 'HELLO')]){
    echo "${env.HELLO} ${env.WORLD}"
    sh "env"
  }
}


