node('centos7-docker-4c-2g'){
  sh 'echo hello'
  sh 'echo Pipeline Triggered by ghprb'
  sh 'echo Pipeline Triggered by ghprb'

  writeFile file:'testFile.txt', text: 'Hello World'

  def fileReadResults = readFile 'testFile.txt'
  echo fileReadResults
}


