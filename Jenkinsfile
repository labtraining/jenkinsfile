// Query outside of node, in order to get pending script approvals
//boolean isTimeTriggered = isTimeTriggeredBuild()

node {

    properties([
            pipelineTriggers(createPipelineTriggers()),
            disableConcurrentBuilds(),
            buildDiscarder(logRotator(numToKeepStr: '10'))
    ])

    catchError {

        stage('Checkout') {
            checkout scm
        }

        stage('Build') {
            mvn 'clean install -DskipTests'
            archiveArtifacts '**/target/*.*ar'
        }

        parallel(
                unitTest: {
                    stage('Unit Test') {
                        mvn 'test'
                    }
                },
                integrationTest: {
                    stage('Integration Test') {
                        if (isNightly()) {
                            mvn 'verify -DskipUnitTests -Parq-wildfly-swarm '
                        }
                    }
                }
        )
    }

    // Archive Unit and integration test results, if any
    junit allowEmptyResults: true,
            testResults: '**/target/surefire-reports/TEST-*.xml, **/target/failsafe-reports/*.xml'

    mailIfStatusChanged env.EMAIL_RECIPIENTS
}

def createPipelineTriggers() {
    if (env.BRANCH_NAME == 'master') {
        // Run a nightly only for master
        return [cron('H H(0-3) * * 1-5')]
    }
    return []
}

/**
 * Determines if nightly build by hour of day to avoid script approval, as need in {@link #isTimeTriggeredBuild()}.
 *
 * @return {@code true} if this build runs between midnight an 3am (within the timezone configured on the Jenkins server).
 */
boolean isNightly() {
    return Calendar.instance.get(Calendar.HOUR_OF_DAY) in 0..3
}

/**
 * Note that this requires the following script approvals by your jenkins administrator
 * (via https://JENKINS-URL/scriptApproval/):
 * <br/>
 * {@code method hudson.model.Run getCauses}
 * {@code method org.jenkinsci.plugins.workflow.support.steps.build.RunWrapper getRawBuild}.
 * <br/><br/>
 * Note that that the pending script approvals only appear if this method is called <b>outside a {@code node}</b>
 * within the pipeline!
 *
 * @return {@code true} if the build was time triggered, otherwise {@code false}
 */
boolean isTimeTriggeredBuild() {
    for (Object currentBuildCause : currentBuild.rawBuild.getCauses()) {
        return currentBuildCause.class.getName().contains('TimerTriggerCause')
    }
    return false
}

def mailIfStatusChanged(String recipients) {
    // Also send "back to normal" emails. Mailer seems to check build result, but SUCCESS is not set at this point.
    if (currentBuild.currentResult == 'SUCCESS') {
        currentBuild.result = 'SUCCESS'
    }
    step([$class: 'Mailer', recipients: recipients])
}

def mvn(def args) {
    def mvnHome = tool 'M3'
    def javaHome = tool 'JDK8'

    // Apache Maven related side notes:
    // --batch-mode : recommended in CI to inform maven to not run in interactive mode (less logs)
    // -V : strongly recommended in CI, will display the JDK and Maven versions in use.
    //      Very useful to be quickly sure the selected versions were the ones you think.
    // -U : force maven to update snapshots each time (default : once an hour, makes no sense in CI).
    // -Dsurefire.useFile=false : useful in CI. Displays test errors in the logs directly (instead of
    //                            having to crawl the workspace files to see the cause).

    // Advice: don't define M2_HOME in general. Maven will autodetect its root fine.
    // See also
    // https://github.com/jenkinsci/pipeline-examples/blob/master/pipeline-examples/maven-and-jdk-specific-version/mavenAndJdkSpecificVersion.groovy
    withEnv(["JAVA_HOME=${javaHome}", "PATH+MAVEN=${mvnHome}/bin:${env.JAVA_HOME}/bin"]) {
        sh "${mvnHome}/bin/mvn ${args} --batch-mode -V -U -e -Dsurefire.useFile=false"
    }
}
