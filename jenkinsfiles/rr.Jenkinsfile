import groovy.json.JsonSlurper

pipeline {
  triggers {
    cron 'H/30 * * * *'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '500', artifactNumToKeepStr: '500'))
    disableConcurrentBuilds()
  }
  agent {
    // Explicitly specify a node, we're depending on the same podman container image being
    // available each time.
    node { }
  }
  parameters {
    string(
      name: 'RUBY_COMMIT',
      description: 'Ruby commit to test',
      defaultValue: 'master',
    )
  }
  environment {
    CONTAINER_IMAGE = "ruby-rr-ci:${env.JEKINS_TAG}"
  }
  stages {
    stage('Prepare SCM') {
      steps {
        dir('ruby') {
          checkout scmGit(
            userRemoteConfigs: [[
              credentialsId: 'github-pat',
              url: 'https://github.com/ruby/ruby.git',
            ]],
            branches: [[name: params.RUBY_COMMIT]],
          )
        }

        script {
          def rubyVersion = sh(script: 'cd ruby; git rev-parse HEAD', returnStdout: true).trim()
          setCustomBuildProperty(key: 'ruby_rr_ci_version', value: "${env.GIT_COMMIT}")
          setCustomBuildProperty(key: 'ruby_version', value: "${rubyVersion}")
          setCustomBuildProperty(key: 'rr', value: "true")
          setCustomBuildProperty(key: 'chaos', value: "false")
          setCustomBuildProperty(key: 'asan', value: "false")
        }
      }
    }
    stage('Build testing image') {
      steps {
        sh 'podman build --tag "${CONTAINER_IMAGE} .'
      }
    }
    stage('Build ruby') {
      steps {
        sh '''
          podman run --rm \
            --mount type=bind,source="$(realpath .)",destination="$(realpath .)",relabel=shared \
            --workdir "$(realpath ./ruby)" \
            --security-opt unmask=/sys/fs/cgroup \
            --security-opt label=disable \
            --security-opt seccomp=unconfined \
            --cgroupns private \
            --userns=keep-id \
            --user "0:0" \
            --env "BUILD_UID=$(id -u)" \
            --env "BUILD_GID=$(id -u)" \
            "${CONTAINER_IMAGE}" \
            ../build-ruby.rb --build
        '''
      }
    }
    stage('Run test suite (btest)') {
      steps {
        sh '''
          podman run --rm \
            --mount type=bind,source="$(realpath .)",destination="$(realpath .)",relabel=shared \
            --workdir "$(realpath ./ruby)" \
            --security-opt unmask=/sys/fs/cgroup \
            --security-opt label=disable \
            --security-opt seccomp=unconfined \
            --cgroupns private \
            --userns=keep-id \
            --user "0:0" \
            --env "BUILD_UID=$(id -u)" \
            --env "BUILD_GID=$(id -u)" \
            "${CONTAINER_IMAGE}" \
            ../build-ruby.rb --btest --rr
        '''
      }
    }
    stage('Run test suite (test-tool)') {
      steps {
        sh '''
          podman run --rm \
            --mount type=bind,source="$(realpath .)",destination="$(realpath .)",relabel=shared \
            --workdir "$(realpath ./ruby)" \
            --security-opt unmask=/sys/fs/cgroup \
            --security-opt label=disable \
            --security-opt seccomp=unconfined \
            --cgroupns private \
            --userns=keep-id \
            --user "0:0" \
            --env "BUILD_UID=$(id -u)" \
            --env "BUILD_GID=$(id -u)" \
            "${CONTAINER_IMAGE}" \
            ../build-ruby.rb --test-tool --rr
        '''
      }
    }
    stage('Run test suite (test-all)') {
      steps {
        sh '''
          podman run --rm \
            --mount type=bind,source="$(realpath .)",destination="$(realpath .)",relabel=shared \
            --workdir "$(realpath ./ruby)" \
            --security-opt unmask=/sys/fs/cgroup \
            --security-opt label=disable \
            --security-opt seccomp=unconfined \
            --cgroupns private \
            --userns=keep-id \
            --user "0:0" \
            --env "BUILD_UID=$(id -u)" \
            --env "BUILD_GID=$(id -u)" \
            "${CONTAINER_IMAGE}" \
            ../build-ruby.rb --test-all --rr
        '''
      }
    }
  }
  post {
    always {
      junit(
       testResults: 'ruby/build/test_output_dir/**/junit.xml',
       testDataPublishers: [[$class:'AttachmentPublisher']],
       keepLongStdio: true,
       allowEmptyResults: true
      )
      sh 'podman image untag "${CONTAINER_IMAGE}"'
    }
  }
}
