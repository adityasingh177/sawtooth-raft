#!groovy

// Copyright 2018 Intel Corporation
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
// ------------------------------------------------------------------------------

// Discard old builds after 31 days
properties([[$class: 'BuildDiscarderProperty', strategy:
        [$class: 'LogRotator', artifactDaysToKeepStr: '',
        artifactNumToKeepStr: '', daysToKeepStr: '31', numToKeepStr: '']]]);

node ('master') {
    timestamps {
        // Create a unique workspace so Jenkins doesn't reuse an existing one
        ws("workspace/${env.BUILD_TAG}") {
            stage("Clone Repo") {
                checkout scm
                sh 'git fetch --tag'
            }

            if (!(env.BRANCH_NAME == 'master' && env.JOB_BASE_NAME == 'master')) {
                stage("Check Whitelist") {
                    readTrusted 'bin/whitelist'
                    sh './bin/whitelist "$CHANGE_AUTHOR" /etc/jenkins-authorized-builders'
                }
            }

            stage("Check for Signed-Off Commits") {
                sh '''#!/bin/bash -l
                    if [ -v CHANGE_URL ] ;
                    then
                        temp_url="$(echo $CHANGE_URL |sed s#github.com/#api.github.com/repos/#)/commits"
                        pull_url="$(echo $temp_url |sed s#pull#pulls#)"

                        IFS=$'\n'
                        for m in $(curl -s "$pull_url" | grep "message") ; do
                            if echo "$m" | grep -qi signed-off-by:
                            then
                              continue
                            else
                              echo "FAIL: Missing Signed-Off Field"
                              echo "$m"
                              exit 1
                            fi
                        done
                        unset IFS;
                    fi
                '''
            }

            // Set the ISOLATION_ID environment variable for the whole pipeline
            env.ISOLATION_ID = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
            env.COMPOSE_PROJECT_NAME = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()

            // Build Raft
            stage("Build Raft") {
              sh "docker-compose up --build --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from raft-engine"
              sh "docker-compose -f docker-compose-installed.yaml build"
            }

            // Run the tests
            stage("Run Tests") {
                sh './bin/run_docker_test tests/test_unit.yaml'
                sh './bin/run_docker_test tests/test_liveness.yaml'
            }

            stage("Build Docs") {
                sh 'docker build . -f docs/Dockerfile -t sawtooth-raft-docs:$ISOLATION_ID'
                sh 'docker run --rm -v $(pwd):/project/sawtooth-raft sawtooth-raft-docs:$ISOLATION_ID'
            }

            stage("Archive Build Artifacts") {
                sh 'docker-compose -f copy-debs.yaml up'
                sh 'docker-compose -f copy-debs.yaml down'
                archiveArtifacts artifacts: 'sawtooth-raft*amd64.deb, docs/build/html/**, docs/build/latex/*.pdf'
            }
        }
    }
}
