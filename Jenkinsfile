#!groovy

def executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER) {
    ansiColor('xterm') {
        echo STAGE_NAME
        ansiblePlaybook (
            installation: 'Default',
            inventory: "${WORKSPACE}/${PipelineProject}/Ansible/hosts.${PipelineProject}",
            playbook: "${WORKSPACE}/${PipelineProject}/Ansible/site.yml",
            sudoUser: null,
            extraVars: [cloud_provider: CLOUD_PROVIDER, ACCOUNT: PROVIDER_PROFILE, stack: PipelineProject, SITE: REGION, pdx_region: REGION, instance_prefix: INSTANCE_PREFIX, SDLC: SDLC, non_parallel: NON_PARALLEL, stack_state: STACK_STATE, target_team: TARGET_TEAM],
            extras: '-f 50 ' + loggingString(params.VERBOSE),
            colorized: true
        )
    }
}

def executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX) {
    ansiColor('xterm') {
        echo STAGE_NAME
        ansiblePlaybook (
            installation: 'Default',
            inventory: "${WORKSPACE}/${PipelineProject}/Ansible/hosts.${PipelineProject}",
            playbook: "${WORKSPACE}/Ansible/tests/site.yml",
            sudoUser: null,
            extraVars: [ SDLC: SDLC, test_hosts: INSTANCE_PREFIX + '-*' ],
            extras: loggingString(params.VERBOSE),
            colorized: true
        )
    }
}

def loggingString(VERBOSE) {
    if (VERBOSE)
        return '-vvv'
    else
        return ''
}

@NonCPS
def ansibleCodePipeline() {
    def job = Jenkins.instance.getItemByFullName('CodePipelines/Ansible')
    def success = job.getLastSuccessfulBuild().getNumber()
    def failure = job.getLastFailedBuild().getNumber()

    if (success > failure) {
        echo "CodePipelines/Ansible pipeline is successful"
        // generate local ansible.cfg if code is passing
        sh 'sed "s~<WORKSPACE>~${WORKSPACE}~g" "${WORKSPACE}/Ansible/ansible.cfg.tmpl" > "${WORKSPACE}/ansible.cfg"'
        return 0
    }
    else {
        error "CodePipelines/Ansible pipeline is currently failing"
        return 1
    }
}

pipeline {
    agent any
    parameters {
        choice(name: 'MAX_SDLC', choices:['Development','QA','Certification','Customer Test','Production'], description:'Environment')
        booleanParam(name: 'VERBOSE', defaultValue: false, description: 'Verbose Logging')
        choice(name: 'CLOUD', choices: ['azure','aws','pdx','all'], description: 'Cloud provider')
        choice(name: 'STACK_STATE', choices: ['present','absent'], description: 'Create/update or delete the stack')
        choice(name: 'TARGET_TEAM', choices: ['all', 'tm1', 'tm2', 'tm10', 'tm14'], description: 'Only apply to a specific team environment (has no effect unless cloud_provider is "all" or "pdx")')
    }
    environment {
        PipelineVersion = VersionNumber([projectStartDate: '2015-12-01', versionNumberString: '${BUILD_YEAR}.${BUILD_MONTH}.${BUILDS_THIS_MONTH}'])
        PipelineProject = "${env.Project}"
        ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
    }
    stages {
        stage('Commit') {
            steps {
                script {
                    ansibleCodePipeline()
                    currentBuild.description = PipelineVersion
                    currentBuild.displayName = PipelineVersion
                    git url: 'scmbuild@git:/srv/gitqa/gitcentral.git'

                    echo "\n------------------ [CommitAnsible] -----------------------"
                    // Perform ansible Syntax checking
                    def params = '--syntax-check --extra-vars "stack=' + PipelineProject + ' instance_prefix=ad pdx_region=us-east-1 SDLC=dev ACCOUNT=TestCS stack_state=' + STACK_STATE + '"'
                    ansiColor('xterm') {
                        ansiblePlaybook (
                            installation: 'Default',
                            inventory: "${WORKSPACE}/${PipelineProject}/Ansible/hosts." + PipelineProject + "",
                            playbook: "${WORKSPACE}/${PipelineProject}/Ansible/site.yml",
                            sudoUser: null,
                            extras: params,
                            colorized: true
                        )
                    }

                    // Execute the ansible-review module to check against best practices
                    def findCmd = 'find ${WORKSPACE}/' + PipelineProject + '/Ansible -type f -name "*.yml"'
                    sh 'set +x;out="";files=`' + findCmd + '`;for i in $files; do res=`ansible-review $i`;  if [ "$res." != . ] ; then out="$out $res" ; fi; done;if [ "$out." != "." ] ; then echo $out; exit 0; fi'

                    // Capture detail to satisfy audits
                    def jenkinsAuditBuildDetails = new com.pdxinc.PDXJenkinsAuditBuildDetails()
                    jenkinsAuditBuildDetails.updateJenkinsAuditBuildDetails("${env.JOB_NAME}-${currentBuild.displayName}")
                    def jenkinsAuditRelDetail = new com.pdxinc.PDXJenkinsAuditReleaseDetails()
                    jenkinsAuditRelDetail.updateJenkinsAuditReleaseDetail("${env.JOB_NAME}-${currentBuild.displayName}","${env.JOB_NAME}-${currentBuild.displayName}")
                }
            }
        }
        stage('Dev') {
            parallel {
                stage('Dev_AWS_East') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Dev_AWS_East'
                        REGION = 'us-east-1'
                        AWS_PROFILE = 'TestCS'
                        INSTANCE_PREFIX = 'ad'
                        SDLC = 'dev'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'

                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Dev_AWS_West') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Dev_AWS_West'
                        REGION = 'us-west-1'
                        AWS_PROFILE = 'TestCS'
                        INSTANCE_PREFIX = 'ad'
                        SDLC = 'dev'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Dev_Azure_SouthCentralUS') {
                    when {
                        expression { return (params.CLOUD == 'azure' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'azure'
                        STAGE_NAME = 'Dev_Azure_SouthCentralUS'
                        REGION = 'southcentralus'
                        AZURE_PROFILE = 'Central Services Test'
                        INSTANCE_PREFIX = 'zd'
                        SDLC = 'dev'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AZURE_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Dev_PDX_DAL') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Dev_PDX_DAL'
                        REGION = 'DAL'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'dd'
                        SDLC = 'dev'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Dev_PDX_FTW') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Dev_PDX_FTW'
                        REGION = 'FTW'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'fd'
                        SDLC = 'dev'
                        NON_PARALLEL = false
                        TARGET_TEAM = ''
                    }
                    steps {
                        script {
                          TARGET_TEAM = params.TARGET_TEAM
                          executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                          executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                        }
                    }
                }
            }
        }
        stage('QA') {
            when {
                expression { return params.MAX_SDLC != 'Development' }
            }
            parallel {
                stage('QA_AWS_East') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'QA_AWS_East'
                        REGION = 'us-east-1'
                        AWS_PROFILE = 'TestCS'
                        INSTANCE_PREFIX = 'aq'
                        SDLC = 'qa'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('QA_AWS_West') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'QA_AWS_West'
                        REGION = 'us-west-1'
                        AWS_PROFILE = 'TestCS'
                        INSTANCE_PREFIX = 'aq'
                        SDLC = 'qa'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('QA_Azure_SouthCentralUS') {
                    when {
                        expression { return (params.CLOUD == 'azure' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'azure'
                        STAGE_NAME = 'QA_Azure_SouthCentralUS'
                        REGION = 'southcentralus'
                        AZURE_PROFILE = 'Central Services Test'
                        INSTANCE_PREFIX = 'zq'
                        SDLC = 'qa'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AZURE_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('QA_PDX_DAL') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'QA_PDX_DAL'
                        REGION = 'DAL'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'dq'
                        SDLC = 'qa'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('QA_PDX_FTW') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'QA_PDX_FTW'
                        REGION = 'FTW'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'fq'
                        SDLC = 'qa'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
            }
        }
        stage('Cert') {
            when {
                expression { return params.MAX_SDLC != 'Development' }
            }
            parallel {
                stage('Cert_AWS_East') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Cert_AWS_East'
                        REGION = 'us-east-1'
                        AWS_PROFILE = 'TestCS'
                        INSTANCE_PREFIX = 'av'
                        SDLC = 'cert'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cert_AWS_West') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Cert_AWS_West'
                        REGION = 'us-west-1'
                        AWS_PROFILE = 'TestCS'
                        INSTANCE_PREFIX = 'av'
                        SDLC = 'cert'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cert_Azure_SouthCentralUS') {
                    when {
                        expression { return (params.CLOUD == 'azure' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'azure'
                        STAGE_NAME = 'Cert_Azure_SouthCentralUS'
                        REGION = 'southcentralus'
                        AZURE_PROFILE = 'Central Services Production'
                        INSTANCE_PREFIX = 'zv'
                        SDLC = 'cert'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AZURE_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cert_PDX_DAL') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Cert_PDX_DAL'
                        REGION = 'DAL'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'dt'
                        SDLC = 'cert'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cert_PDX_FTW') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Cert_PDX_FTW'
                        REGION = 'FTW'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'ft'
                        SDLC = 'cert'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
            }
        }
        stage('Cust') {
            when {
                expression { return (params.MAX_SDLC == 'Customer Test' || params.MAX_SDLC == 'Production') }
            }
            parallel {
                stage('Cust_AWS_East') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Cust_AWS_East'
                        REGION = 'us-east-1'
                        AWS_PROFILE = 'ProdCS'
                        INSTANCE_PREFIX = 'at'
                        SDLC = 'cust'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cust_AWS_West') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Cust_AWS_West'
                        REGION = 'us-west-1'
                        AWS_PROFILE = 'ProdCS'
                        INSTANCE_PREFIX = 'at'
                        SDLC = 'cust'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cust_Azure_SouthCentralUS') {
                    when {
                        expression { return (params.CLOUD == 'azure' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'azure'
                        STAGE_NAME = 'Cust_Azure_SouthCentralUS'
                        REGION = 'southcentralus'
                        AZURE_PROFILE = 'Central Services Production'
                        INSTANCE_PREFIX = 'zt'
                        SDLC = 'cust'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AZURE_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cust_PDX_DAL') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Cust_PDX_DAL'
                        REGION = 'DAL'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'dt'
                        SDLC = 'cust'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Cust_PDX_FTW') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Cust_PDX_FTW'
                        REGION = 'FTW'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'ft'
                        SDLC = 'cust'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
            }
        }
        stage('Prod') {
            when {
                expression { return params.MAX_SDLC == 'Production' }
            }
            parallel {
                stage('Prod_AWS_East') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Prod_AWS_East'
                        REGION = 'us-east-1'
                        AWS_PROFILE = 'ProdCS'
                        INSTANCE_PREFIX = 'ap'
                        SDLC = 'prod'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Prod_AWS_West') {
                    when {
                        expression { return (params.CLOUD == 'aws' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'aws'
                        STAGE_NAME = 'Prod_AWS_West'
                        REGION = 'us-west-1'
                        AWS_PROFILE = 'ProdCS'
                        INSTANCE_PREFIX = 'ap'
                        SDLC = 'prod'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AWS_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Prod_Azure_SouthCentralUS') {
                    when {
                        expression { return (params.CLOUD == 'azure' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'azure'
                        STAGE_NAME = 'Prod_Azure_SouthCentralUS'
                        REGION = 'southcentralus'
                        AZURE_PROFILE = 'Central Services Production'
                        INSTANCE_PREFIX = 'zp'
                        SDLC = 'prod'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, AZURE_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Prod_PDX_DAL') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Prod_PDX_DAL'
                        REGION = 'DAL'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'dp'
                        SDLC = 'prod'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
                stage('Prod_PDX_FTW') {
                    when {
                        expression { return (params.CLOUD == 'pdx' || params.CLOUD == 'all') }
                    }
                    environment {
                        CLOUD_PROVIDER = 'pdx'
                        STAGE_NAME = 'Prod_PDX_FTW'
                        REGION = 'FTW'
                        PROVIDER_PROFILE = 'NA'
                        INSTANCE_PREFIX = 'fp'
                        SDLC = 'prod'
                        NON_PARALLEL = false
                        TARGET_TEAM = 'NA'
                    }
                    steps {
                        executeAnsiblePlaybook(STAGE_NAME, REGION, PROVIDER_PROFILE, INSTANCE_PREFIX, SDLC, NON_PARALLEL, STACK_STATE, TARGET_TEAM, CLOUD_PROVIDER)
                        executeComplianceTests(STAGE_NAME, SDLC, INSTANCE_PREFIX)
                    }
                }
            }
        }
    }
    post {
        always {
            echo "\nBuild ${PipelineProject}:${PipelineVersion}: SUCCESS"
        }
        aborted {
            echo "\nBuild ${PipelineProject}:${PipelineVersion}: ABORTED"
        }
    }
}
