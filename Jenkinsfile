node {
  def commit_id

  def to = emailextrecipients([
      [$class: 'CulpritsRecipientProvider'],
      [$class: 'DevelopersRecipientProvider'],
      [$class: 'RequesterRecipientProvider']
  ])

  try {
    stage('Preparation') {
      checkout scm
      sh "git rev-parse --short HEAD > .git/commit-id"
      commit_id = readFile('.git/commit-id').trim()
    }

    stage('test') {
      def myTestContainer = docker.image('node:9.1')

      myTestContainer.pull()

      myTestContainer.inside {
        sh 'npm install --only=dev'
        sh 'npm test'
      }
    }

    stage('test with a DB') {
      def mysql = docker.image('mysql').run("-e MYSQL_ALLOW_EMPTY_PASSWORD=yes")

      def myTestContainer = docker.image('node:9.1')

      myTestContainer.pull()

      myTestContainer.inside("--link ${mysql.id}:mysql") {
        // using linking, mysql will be available at host: mysql, port: 3306
        sh 'npm install --only=dev'
        sh 'npm test'
      }

      mysql.stop()
    }

    stage('docker build/push') {
      docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
        def app = docker.build("davidainslie/docker-nodejs-demo:${commit_id}", '.').push()
      }
    }
  } catch(e) {
    // mark build as failed
    currentBuild.result = "FAILURE";
    
    // set variables
    def subject = "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} ${currentBuild.result}"
    def content = '${JELLY_SCRIPT,template="html"}'

    // send email
    if (to != null && !to.isEmpty()) {
      emailext(body: content, mimeType: 'text/html',
          replyTo: '$DEFAULT_REPLYTO', subject: subject,
          to: to, attachLog: true )
    }

    // send slack notification
    slackSend(color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")

    // mark current build as a failure and throw the error
    throw e;
  }
}