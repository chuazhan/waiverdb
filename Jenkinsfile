/*
 * SPDX-License-Identifier: GPL-2.0+
*/
import groovy.json.*

// 'global' var to store git info
def scmVars

try { // massive try{} catch{} around the entire build for failure notifications

node('master'){
    scmVars = checkout scm
    scmVars.GIT_BRANCH_NAME = scmVars.GIT_BRANCH.split('/')[-1]  // origin/pr/1234 -> 1234

    // setting build display name
    def branch = scmVars.GIT_BRANCH_NAME
    if ( branch == 'master' ) {
        echo 'Building master'
        currentBuild.displayName = 'master'
    }
    else if (branch ==~ /[0-9]+/) {
        def pagureUrl = "https://pagure.io/waiverdb/pull-request/${branch}"
        def pagureLink = """<a href="${pagureUrl}">PR-${branch}</a>"""
        try {
            def response = httpRequest "https://pagure.io/api/0/waiverdb/pull-request/${branch}"
            // Note for future use: JsonSlurper() is not serialiazble (returns a LazyMap) and
            // therefore we cannot save this back into the global scmVars. We could use
            // JsonSlurperClassic() which returns a hash map, but would need to allow this in
            // the jenkins script approval.
            def content = new JsonSlurper().parseText(response.content)
            pagureLink = """<a href="${pagureUrl}">${content.title}</a>"""
        } catch (Exception e) {
            echo 'Error using pagure API:'
            echo e.message
            // ignoring this...
        }
        echo "Building PR #${branch}: ${pagureUrl}"
        currentBuild.displayName = "PR #${branch}"
        currentBuild.description = pagureLink
    }
}

timestamps {

node('fedora-29') {
    checkout scm
    stage('Prepare') {
        sh 'sudo dnf -y builddep waiverdb.spec'
        sh 'sudo dnf -y install python3-flake8 python3-pylint python3-pytest python3-sphinx python3-sphinxcontrib-httpdomain'
        /* Needed for mock EPEL7 builds: https://bugzilla.redhat.com/show_bug.cgi?id=1528272 */
        sh 'sudo dnf -y install dnf-utils'
        /* Needed to get the latest /etc/mock/fedora-28-x86_64.cfg */
        sh 'sudo dnf -y update mock-core-configs'
        /* Unit tests need local Postgres */
        sh """
        sudo dnf -y remove postgresql-server || true
        sudo rm -rf /var/lib/pgsql/ || true
        sudo dnf -y install postgresql-server
        sudo postgresql-setup --initdb
        sudo systemctl enable --now postgresql
        sudo -u postgres createuser --superuser $USER
        """
    }
    stage('Invoke Flake8') {
        sh 'flake8'
    }
    stage('Invoke Pylint') {
        sh 'pylint-3 --reports=n waiverdb'
    }
    stage('Run unit tests') {
        sh """
        createdb waiverdb
        cp conf/settings.py.example conf/settings.py
        py.test-3 -v --junitxml=junit-tests.xml tests/
        """
        junit 'junit-tests.xml'
    }
    stage('Build Docs') {
        sh 'make -C docs html'
        archiveArtifacts artifacts: 'docs/_build/html/**'
    }
    stage('Build SRPM') {
        sh './rpmbuild.sh -bs'
        archiveArtifacts artifacts: 'rpmbuild-output/**'
    }
    /* We take a flock on the mock configs, to avoid multiple unrelated jobs on
     * the same Jenkins slave trying to use the same mock root at the same
     * time, which will error out. */
    stage('Build RPM') {
        parallel (
            'EPEL7': {
                sh """
                mkdir -p mock-result/el7
                flock /etc/mock/epel-7-x86_64.cfg \
                /usr/bin/mock -v --resultdir=mock-result/el7 -r epel-7-x86_64 --clean --rebuild rpmbuild-output/*.src.rpm
                """
                archiveArtifacts artifacts: 'mock-result/el7/**'
            },
            'F29': {
                sh """
                mkdir -p mock-result/f29
                flock /etc/mock/fedora-29-x86_64.cfg \
                /usr/bin/mock -v --resultdir=mock-result/f29 -r fedora-29-x86_64 --clean --rebuild rpmbuild-output/*.src.rpm
                """
                archiveArtifacts artifacts: 'mock-result/f29/**'
            },
        )
    }
    stage('Invoke Rpmlint') {
        parallel (
            'EPEL7': {
                sh 'rpmlint -f rpmlint-config.py mock-result/el7/*.rpm'
            },
            'F29': {
                sh 'rpmlint -f rpmlint-config.py mock-result/f29/*.rpm'
            },
        )
    }
}
node('docker') {
    checkout scm
    stage('Build Docker container') {
        unarchive mapping: ['mock-result/f29/': '.']
        def f29_rpm = findFiles(glob: 'mock-result/f29/**/*.noarch.rpm')[0]
        /* XXX: remove this once we divorce waiverdb-cli from waiverdb. */
        def waiverdb_common = findFiles(glob: 'mock-result/f29/**/waiverdb-common-*.noarch.rpm')[0]
        def appversion = sh(returnStdout: true, script: """
            rpm2cpio ${waiverdb_common } | \
            cpio --quiet --extract --to-stdout ./usr/lib/python\\*/site-packages/waiverdb\\*.egg-info/PKG-INFO | \
            awk '/^Version: / {print \$2}'
        """).trim()
        /* Git builds will have a version like 0.3.2.dev1+git.3abbb08 following
         * the rules in PEP440. But Docker does not let us have + in the tag
         * name, so let's munge it here. */
        appversion = appversion.replace('+', '-')
        docker.withRegistry(
                'https://quay.io/',
                'quay-io-factory2-builder-sa-credentials') {
            def image = docker.build "factory2/waiverdb:${appversion}", "--build-arg waiverdb_rpm=$f29_rpm --build-arg waiverdb_common_rpm=$waiverdb_common ."
            image.push()
        }
        /* Save container version for later steps (this is ugly but I can't find anything better...) */
        writeFile file: 'appversion', text: "${appversion}"
        archiveArtifacts artifacts: 'appversion'
    }
}
node('fedora-29') {
    sh 'sudo dnf -y install /usr/bin/py.test-3'
    checkout scm
    stage('Perform functional tests') {
        unarchive mapping: ['appversion': 'appversion']
        def appversion = readFile('appversion').trim()
        openshift.withCluster('Upshift-GA') {
            openshift.doAs('upshift-waiverdb-test-jenkins-credentials') {
                openshift.withProject('waiverdb-test') {
                    def template = readYaml file: 'openshift/waiverdb-test-template.yaml'
                    def models = openshift.process(template,
                            '-p', "TEST_ID=${env.BUILD_TAG}",
                            '-p', "WAIVERDB_APP_IMAGE=quay.io/factory2/waiverdb:internal-${appversion}")
                    def environment_label = "test-${env.BUILD_TAG}"
                    try {
                        def objects = openshift.create(models)
                        echo "Waiting for pods with label environment=${environment_label} to become Ready"
                        def pods = openshift.selector('pods', ['environment': environment_label])
                        timeout(30) {
                            pods.untilEach(3) {
                                def conds = it.object().status.conditions
                                for (int i = 0; i < conds.size(); i++) {
                                    if (conds[i].type == 'Ready' && conds[i].status == 'True') {
                                        return true
                                    }
                                }
                                return false
                            }
                        }
                        def route_hostname = objects.narrow('route').object().spec.host
                        echo "Running tests against https://${route_hostname}/"
                        def ca_chain = sh(returnStdout: true, script: """openssl s_client \
                                -connect ${route_hostname}:443 \
                                -servername ${route_hostname} -showcerts < /dev/null | \
                              awk 'BEGIN {first_cert=1; in_cert=0};
                                   /BEGIN CERTIFICATE/ { if (first_cert == 1) first_cert = 0; else in_cert = 1 };
                                   { if (in_cert) print };
                                   /END CERTIFICATE/ { in_cert = 0 }'""")
                        writeFile(file: "${env.WORKSPACE}/ca-chain.crt", text: ca_chain)
                        echo "Wrote CA certificate chain to ${env.WORKSPACE}/ca-chain.crt"
                        withEnv(["WAIVERDB_TEST_URL=https://${route_hostname}/",
                                 "REQUESTS_CA_BUNDLE=${env.WORKSPACE}/ca-chain.crt"]) {
                            sh 'py.test-3 -v --junitxml=junit-functional-tests.xml functional-tests/'
                        }
                        junit 'junit-functional-tests.xml'
                    } finally {
                        /* Extract logs for debugging purposes */
                        openshift.selector('deploy,pods', ['environment': environment_label]).logs()
                        /* Tear down everything we just created */
                        openshift.selector('dc,deploy,configmap,secret,svc,route',
                                ['environment': environment_label]).delete()
                    }
                }
            }
        }
    }
}
    } // end timestamps
} catch (e) {
    // since the result isn't set until after the pipeline script runs, we must set it here if it fails
    currentBuild.result = 'FAILURE'
    throw e
} finally {
    // if result hasn't been set to failure by this point, its a success.
    def currentResult = currentBuild.result ?: 'SUCCESS'
    def branch = scmVars.GIT_BRANCH_NAME

    // send pass/fail email
    def SUBJECT = ''
    if ( branch ==~ /[0-9]+/) {
        if (currentResult == 'FAILURE' ){
            SUBJECT = "Jenkins job ${env.JOB_NAME}, PR #${branch} failed."
        } else {
            SUBJECT = "Jenkins job ${env.JOB_NAME}, PR #${branch} passed."
        }
    } else if (currentResult == 'FAILURE') {
        SUBJECT = "Jenkins job ${env.JOB_NAME} #${env.BUILD_NUMBER} failed."
    }

    def RECIEPENT = scmVars.GIT_AUTHOR_EMAIL
    if (ownership.job.ownershipEnabled && branch == 'master') {
        RECIEPENT = ownership.job.primaryOwnerEmail
    }

    def BODY = "Build URL: ${env.BUILD_URL}"
    if (branch ==~ /[0-9]+/){
        BODY = BODY + "\nPull Request: https://pagure.io/waiverdb/pull-request/${branch}"
    }

    if (SUBJECT != '') {
        emailext to: RECIEPENT,
                 subject: SUBJECT,
                 body: BODY
    }

    // update Pagure PR status
    if (branch ==~ /[0-9]+/) {  // PR's will only be numbers on pagure
        def resultPercent = (currentResult == 'SUCCESS') ? '100' : '0'
        def resultComment = (currentResult == 'SUCCESS') ? 'Build passed.' : 'Build failed.'
        def pagureRepo = new URL(scmVars.GIT_URL).getPath() - ~/^\// - ~/.git$/  // https://pagure.io/my-repo.git -> my-repo

        withCredentials([string(credentialsId: "${env.PAGURE_API_TOKEN}", variable: 'TOKEN')]) {
        build job: 'pagure-PR-status-updater',
            propagate: false,
            parameters: [
                // [$class: 'StringParameterValue', name: 'PAGURE_REPO', value: 'https://pagure.io'],  // not needed if https://pagure.io
                [$class: 'StringParameterValue', name: 'PAGURE_PR', value: branch],
                [$class: 'StringParameterValue', name: 'PAGURE_REPO', value: pagureRepo],
                [$class: 'StringParameterValue', name: 'PERCENT_PASSED', value: resultPercent],
                [$class: 'StringParameterValue', name: 'COMMENT', value: resultComment],
                [$class: 'StringParameterValue', name: 'REFERENCE_URL', value: "${env.BUILD_URL}"],
                [$class: 'StringParameterValue', name: 'REFERENCE_JOB_NAME', value: "${env.JOB_NAME}"],
                [$class: 'hudson.model.PasswordParameterValue', name: 'TOKEN', value: "${env.TOKEN}"]
                        ]
        }
    }
}
