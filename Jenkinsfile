pipeline {
  agent { label 'MacMini' }
  options { skipDefaultCheckout() }
  parameters {
    gitParameter(name: 'GIT_TAG',
                 type: 'PT_BRANCH_TAG',
                 description: 'The Git tag to checkout. If not specified "main" will be checkout.',
                 defaultValue: 'master')
    string(name: 'RUST_TOOLCHAIN',
           description: 'The version of rust toolchain to use (e.g. nightly-2020-12-20)',
           defaultValue: 'nightly')
    booleanParam(name: 'BUILD_MACOSX',
                 description: 'Build macosx target.',
                 defaultValue: true)
    booleanParam(name: 'BUILD_DOCKER',
                 description: 'Build for janu in Docker (Alpine x86_64-unknown-linux-musl target).',
                 defaultValue: true)
    booleanParam(name: 'BUILD_LINUX64',
                 description: 'Build x86_64-unknown-linux-gnu target.',
                 defaultValue: true)
    booleanParam(name: 'BUILD_LINUX32',
                 description: 'Build i686-unknown-linux-gnu target.',
                 defaultValue: true)
    booleanParam(name: 'BUILD_AARCH64',
                 description: 'Build aarch64-unknown-linux-gnu target.',
                 defaultValue: true)
    booleanParam(name: 'BUILD_WIN64',
                 description: 'Build x86_64-pc-windows-gnu target.',
                 defaultValue: true)
    booleanParam(name: 'BUILD_WIN32',
                 description: 'Build i686-pc-windows-gnu target.',
                 defaultValue: true)
    booleanParam(name: 'PUBLISH_ECLIPSE_DOWNLOAD',
                 description: 'Publish the resulting artifacts to Eclipse download.',
                 defaultValue: false)
  }
  environment {
      LABEL = get_label()
      DOWNLOAD_DIR="/home/data/httpd/download.eclipse.org/janu/janu-backend-filesystem/${LABEL}"
      MACOSX_DEPLOYMENT_TARGET=10.9
  }

  stages {
    stage('Checkout Git TAG') {
      steps {
        deleteDir()
        checkout([$class: 'GitSCM',
                  branches: [[name: "${params.GIT_TAG}"]],
                  doGenerateSubmoduleConfigurations: false,
                  extensions: [],
                  gitTool: 'Default',
                  submoduleCfg: [],
                  userRemoteConfigs: [[url: 'https://github.com/virtuehive/janu-backend-filesystem.git']]
                ])
      }
    }
    stage('Update Rust env') {
      steps {
        sh '''
        env
        rustup default ${RUST_TOOLCHAIN}
        '''
      }
    }

    stage('MacOS build') {
      when { expression { return params.BUILD_MACOSX }}
      steps {
        sh '''
        echo "Building janu-backend-filesystem-${LABEL}"
        cargo build --release
        cargo test --release
        tar -czvf janu-backend-filesystem-${LABEL}-macosx${MACOSX_DEPLOYMENT_TARGET}-x86-64.tgz --strip-components 2 target/release/*.dylib
        '''
      }
    }

    stage('x86_64-unknown-linux-musl build') {
      when { expression { return params.BUILD_DOCKER }}
      steps {
        sh '''
        docker run --init --rm -v $(pwd):/workdir -w /workdir adlinktech/janu-dev-x86_64-unknown-linux-musl \
          /bin/ash -c "\
            rustup default ${RUST_TOOLCHAIN} && \
            cargo build --release \
          "
        tar -czvf janu-backend-filesystem-${LABEL}-x86_64-unknown-linux-musl.tgz --strip-components 3 target/x86_64-unknown-linux-musl/release/*.so
        '''
      }
    }

    stage('x86_64-unknown-linux-gnu build') {
      when { expression { return params.BUILD_LINUX64 }}
      steps {
        sh '''
        docker run --init --rm -v $(pwd):/workdir -w /workdir adlinktech/janu-dev-manylinux2010-x86_64-gnu \
          /bin/bash -c "\
            rustup default ${RUST_TOOLCHAIN} && \
            cargo build --release && \
            if [[ ${GIT_TAG} != origin/* ]]; then \
                cargo deb \
            ;fi \
          "
        tar -czvf janu-backend-filesystem-${LABEL}-x86_64-unknown-linux-gnu.tgz --strip-components 3 target/x86_64-unknown-linux-gnu/release/*.so
        '''
      }
    }

    stage('i686-unknown-linux-gnu build') {
      when { expression { return params.BUILD_LINUX32 }}
      steps {
        sh '''
        docker run --init --rm -v $(pwd):/workdir -w /workdir adlinktech/janu-dev-manylinux2010-i686-gnu \
          /bin/bash -c "\
            rustup default ${RUST_TOOLCHAIN} && \
            cargo build --release && \
            if [[ ${GIT_TAG} != origin/* ]]; then \
                cargo deb \
            ;fi \
          "
        tar -czvf janu-backend-filesystem-${LABEL}-i686-unknown-linux-gnu.tgz --strip-components 3 target/i686-unknown-linux-gnu/release/*.so
        '''
      }
    }

    stage('aarch64-unknown-linux-gnu build') {
      when { expression { return params.BUILD_AARCH64 }}
      steps {
        sh '''
        docker run --init --rm -v $(pwd):/workdir -w /workdir adlinktech/janu-dev-manylinux2014-aarch64-gnu \
          /bin/bash -c "\
            rustup default ${RUST_TOOLCHAIN} && \
            cargo build --release && \
            if [[ ${GIT_TAG} != origin/* ]]; then \
                cargo deb \
            ;fi \
          "
        tar -czvf janu-backend-filesystem-${LABEL}-aarch64-unknown-linux-gnu.tgz --strip-components 3 target/aarch64-unknown-linux-gnu/release/*.so
        '''
      }
    }

    stage('x86_64-pc-windows-gnu build') {
      when { expression { return params.BUILD_WIN64 }}
      steps {
        sh '''
        cargo build --release --bins --lib --examples --target=x86_64-pc-windows-gnu
        zip janu-backend-filesystem-${LABEL}-x86_64-pc-windows-gnu.zip --junk-paths target/x86_64-pc-windows-gnu/release/*.dll
        '''
      }
    }

    stage('i686-pc-windows-gnu build') {
      when { expression { return params.BUILD_WIN32 }}
      steps {
        sh '''
        cargo build --release --bins --lib --examples --target=i686-pc-windows-gnu
        zip janu-backend-filesystem-${LABEL}-i686-pc-windows-gnu.zip --junk-paths target/i686-pc-windows-gnu/release/*.dll
        '''
      }
    }

    stage('Prepare directory on download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD }}
      steps {
        // Note: remove existing dir on download.eclipse.org only if it's for a branch
        // (e.g. master that is rebuilt periodically from different commits)
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            if [[ ${GIT_TAG} == origin/* ]]; then
              ssh genie.janu@projects-storage.eclipse.org rm -fr ${DOWNLOAD_DIR}
            fi
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            COMMIT_ID=`git log -n1 --format="%h"`
            echo "https://github.com/virtuehive/janu-backend-filesystem/tree/${COMMIT_ID}" > _git_commit_${COMMIT_ID}.txt
            rustc --version > _rust_toolchain_${RUST_TOOLCHAIN}.txt
            scp _*.txt genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
          '''
        }
      }
    }

    stage('Publish janu-macosx to download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD && params.BUILD_MACOSX }}
      steps {
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            scp janu-backend-filesystem-${LABEL}-*macosx*.tgz genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
          '''
        }
      }
    }

    stage('Publish janu-x86_64-unknown-linux-musl to download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD && params.BUILD_DOCKER }}
      steps {
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            scp janu-backend-filesystem-${LABEL}-x86_64-unknown-linux-musl.tgz genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
          '''
        }
      }
    }

    stage('Publish janu-x86_64-unknown-linux-gnu to download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD && params.BUILD_LINUX64 }}
      steps {
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            scp janu-backend-filesystem-${LABEL}-x86_64-unknown-linux-gnu.tgz genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
            if [[ ${GIT_TAG} != origin/* ]]; then
              scp target/x86_64-unknown-linux-gnu/debian/*.deb genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
            fi
          '''
        }
      }
    }

    stage('Publish janu-i686-unknown-linux-gnu to download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD && params.BUILD_LINUX32 }}
      steps {
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            scp janu-backend-filesystem-${LABEL}-i686-unknown-linux-gnu.tgz genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
            if [[ ${GIT_TAG} != origin/* ]]; then
              scp target/i686-unknown-linux-gnu/debian/*.deb genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
            fi
          '''
        }
      }
    }

    stage('Publish janu-x86_64-pc-windows-gnu to download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD && params.BUILD_WIN64 }}
      steps {
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            scp janu-backend-filesystem-${LABEL}-x86_64-pc-windows-gnu.zip genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
          '''
        }
      }
    }

    stage('Publish janu-i686-pc-windows-gnu to download.eclipse.org') {
      when { expression { return params.PUBLISH_ECLIPSE_DOWNLOAD && params.BUILD_WIN32 }}
      steps {
        sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
          sh '''
            ssh genie.janu@projects-storage.eclipse.org mkdir -p ${DOWNLOAD_DIR}
            scp janu-backend-filesystem-${LABEL}-i686-pc-windows-gnu.zip genie.janu@projects-storage.eclipse.org:${DOWNLOAD_DIR}/
          '''
        }
      }
    }

  }
}

def get_label() {
    return env.GIT_TAG.startsWith('origin/') ? env.GIT_TAG.minus('origin/') : env.GIT_TAG
}
