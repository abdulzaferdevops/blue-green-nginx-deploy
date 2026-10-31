// Lab 7: blue-green-nginx-deploy
// Blue-green deployment of the Lab 4 image (by digest) behind nginx on the app server.
//
//   status   = show both slots (the default, so a push to this repo changes nothing)
//   deploy   = start IMAGE_DIGEST in the idle slot, health-check it, test it on the preview
//              port, then move live traffic to it, counting failed requests during the switch
//   rollback = move live traffic back to the previous release, which is still running
//
// Uses the ci-labs shared library (Lab 6).

@Library('ci-labs@v1.0.0') _

pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 20, unit: 'MINUTES')
  }

  triggers {
    githubPush()
  }

  parameters {
    choice(name: 'ACTION', choices: ['status', 'deploy', 'rollback'],
           description: 'status = show the slots; deploy = blue-green release of IMAGE_DIGEST; rollback = return live traffic to the previous release')
    string(name: 'IMAGE_DIGEST', defaultValue: '', trim: true,
           description: 'For deploy: <registry>/<namespace>/ci-labs/flask-api@sha256:<64 hex> (from Lab 4 image.json)')
    booleanParam(name: 'APPROVE_SWITCH', defaultValue: false,
                 description: 'Pause for a person to approve before live traffic moves')
    booleanParam(name: 'SIMULATE_FAILURE', defaultValue: false,
                 description: 'Lab demo: make the idle-slot health check expect the wrong release, to show a failing release never receives traffic')
  }

  stages {
    stage('Preflight') {
      steps {
        script {
          labPreflight(vars: ['APP_HOST', 'OCIR_HOST', 'OCIR_NAMESPACE'])
          env.REPO = "${env.OCIR_HOST}/${env.OCIR_NAMESPACE}/ci-labs/flask-api"
          env.ACTION_NAME = params.ACTION ?: 'status'
          currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.ACTION_NAME}"
          labSsh(command: 'echo "connected to $(hostname -s) as $(whoami)"')
          labSsh(script: 'scripts/release.sh', args: ['status'], label: 'slots before')
        }
      }
    }

    stage('Resolve image') {
      when { expression { env.ACTION_NAME == 'deploy' } }
      steps {
        script {
          if (!params.IMAGE_DIGEST?.trim()) {
            error('deploy needs IMAGE_DIGEST: paste the digest from Lab 4 image.json')
          }
          env.TARGET_IMAGE = labImageRef(repo: env.REPO, ref: params.IMAGE_DIGEST.trim())
          String t = env.TARGET_IMAGE
          currentBuild.displayName = "#${env.BUILD_NUMBER} deploy ${t.substring(t.length() - 12)}"
        }
      }
    }

    stage('Pull on server') {
      when { expression { env.ACTION_NAME == 'deploy' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'ocir-push',
                                          usernameVariable: 'OCIR_USER', passwordVariable: 'OCIR_TOKEN')]) {
          sshagent(credentials: ['app-server-deploy']) {
            // Same as Lab 5: the token travels on stdin into an owner-only temporary auth file
            // that is deleted after the pull. TARGET_IMAGE was digest-validated above.
            sh '''
              printf '%s' "$OCIR_TOKEN" | ssh -o BatchMode=yes -o ConnectTimeout=10 deploy@$APP_HOST \
                "umask 077; D=\\$(mktemp -d); A=\\$D/auth.json; trap 'rm -rf \\$D' EXIT; podman login --authfile \\$A -u '$OCIR_USER' --password-stdin $OCIR_HOST >/dev/null && podman pull --quiet --authfile \\$A '$TARGET_IMAGE' >/dev/null && echo 'pulled: $TARGET_IMAGE (temporary login removed)'"
            '''
          }
        }
      }
    }

    stage('Stage idle slot') {
      when { expression { env.ACTION_NAME == 'deploy' } }
      steps {
        script {
          labSsh(script: 'scripts/release.sh',
                 args: ['stage', env.TARGET_IMAGE, params.SIMULATE_FAILURE ? '1' : '0'],
                 label: 'stage in the idle slot')
          String staged = labSsh(script: 'scripts/release.sh', args: ['staged'], returnStdout: true)
          List parts = staged.tokenize(' ')
          if (parts.size() != 2) { error("could not read the staged slot: '${staged}'") }
          env.NEW_SLOT = parts[0]
          env.NEW_RELEASE = parts[1]
          echo "staged ${env.NEW_RELEASE} in ${env.NEW_SLOT}"
        }
      }
    }

    stage('Preview test') {
      when { expression { env.ACTION_NAME == 'deploy' } }
      steps {
        script {
          labSsh(script: 'scripts/release.sh', args: ['preview', env.NEW_SLOT, env.NEW_RELEASE],
                 label: 'test through the preview port')
        }
      }
    }

    stage('Approve switch') {
      when { expression { env.ACTION_NAME == 'deploy' && params.APPROVE_SWITCH } }
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          input(message: "Move live traffic on :8087 to ${env.NEW_SLOT} (${env.NEW_RELEASE})?", ok: 'Switch')
        }
      }
    }

    stage('Switch') {
      when { expression { env.ACTION_NAME == 'deploy' } }
      steps {
        script {
          labSsh(script: 'scripts/release.sh', args: ['switch', env.NEW_SLOT, env.NEW_RELEASE],
                 label: 'switch live traffic')
        }
      }
    }

    stage('Rollback') {
      when { expression { env.ACTION_NAME == 'rollback' } }
      steps {
        script {
          labSsh(script: 'scripts/release.sh', args: ['rollback'], label: 'rollback')
        }
      }
    }

    stage('Downtime check') {
      when { expression { env.ACTION_NAME in ['deploy', 'rollback'] } }
      steps {
        script {
          String line = labSsh(script: 'scripts/release.sh', args: ['last-switch'], returnStdout: true)
          echo "last switch: ${line}"
          String failedText = ''
          for (String word : line.tokenize(' ')) {
            if (word.startsWith('failed=')) { failedText = word.substring(7) }
          }
          if (!(failedText ==~ /\d+/)) { error("no switch result recorded: '${line}'") }
          int failed = Integer.parseInt(failedText)
          currentBuild.description = line.replaceAll(/ \(.*\)/, '')
          if (failed > 0) {
            unstable("${failed} request(s) failed while live traffic moved")
          } else {
            echo 'zero failed requests during the switch'
          }
        }
      }
    }
  }

  post {
    always {
      script {
        try {
          labSsh(script: 'scripts/release.sh', args: ['status'], label: 'slots after')
        } catch (err) {
          echo "could not read the slots: ${err.message}"
        }
      }
    }
  }
}
