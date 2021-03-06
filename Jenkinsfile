@Library(['rhv-qe-jenkins-library@master']) _

properties(
  [
    parameters(
      [
        string(defaultValue: 'v2v-node-rdu', description: 'Name or label of slave to run on', name: 'NODE_LABEL'),
        choice(defaultValue: 'RDU', description: 'Choose the environment', name: 'ENVIRONMENT', choices: ['RDU', 'TLV']),
        booleanParam(defaultValue: false, description: 'Recreating RHV Storage Domain. Supported only in RDU environment', name: 'RHV_RECREATE_STORAGE'),
        booleanParam(defaultValue: false, description: 'Nightly pre check', name: 'MIQ_NIGHTLY_PRE_CHECK'),
        booleanParam(defaultValue: false, description: 'Remove existing instance', name: 'MIQ_REMOVE_EXISTING_INSTANCE'),
        string(defaultValue: '', description: 'Name of GE or label that match the desired GE.', name: 'GE_NAME'),
        string(defaultValue: '', description: 'The name of the main YAML file e.g. v2v-1. The file placed under rhevm-jenkins/qe/v2v/', name: 'SOURCE_YAML'),
        string(defaultValue: '', description: 'Image URL e.g. http://file.cloudforms.lab.eng.rdu2.redhat.com/builds/cfme/5.10/stable/cfme-rhevm-5.10.0.33-1.x86_64.qcow2', name: 'CFME_IMAGE_URL'),
        string(defaultValue: '', description: 'RHV hosts selection, separated by a comma e.g. 1,3-5,7. Leave empty to use ALL hosts', name: 'RHV_HOSTS'),
        string(defaultValue: '', description: 'VMware hosts selection, separated by a comma e.g. 1,3-5,7. Leave empty to use ALL hosts', name: 'VMW_HOSTS'),
        choice(defaultValue: 'iscsi', description: 'Choose the target RHV storage type. This parameter is relevant only for RDU environment', name: 'RHV_STORAGE', choices: ['iscsi', 'fcp']),
        string(defaultValue: '', description: 'The source VMware data storage name. If left empty, the name will be set accordingly to source YML file', name: 'VMW_STORAGE_NAME'),
        string(defaultValue: '', description: 'The target RHV data storage name. If left empty, the name will be set accordingly to source YML file', name: 'RHV_STORAGE_NAME'),
        string(defaultValue: '', description: 'The number of hosts to be migrated', name: 'NUMBER_OF_VMS'),
        string(defaultValue: '', description: 'VMware Template name', name: 'VMW_TEMPLATE_NAME'),
        choice(defaultValue: 'VDDK', description: 'Migration Protocol - SSH/VDDK', name: 'TRANSPORT_METHODS', choices: ['VDDK', 'SSH']),
        string(defaultValue: '20', description: 'Provider concurrent migration max num of VMs', name: 'PROVIDER_CONCURRENT_MAX'),
        string(defaultValue: '10', description: 'Host concurrent migration max num of VMs', name: 'HOST_CONCURRENT_MAX'),
        choice(defaultValue: 'Create VMs', description: 'Specify a stage to run from', name: 'START_FROM_STAGE', choices: ['Create VMs', 'Remove RHV VMs', 'Install Nmon', 'Add extra providers', 'Set RHV provider concurrent VM migration max', 'Configure oVirt conversion hosts', 'Configure ESX hosts', 'SSH Configuration', 'Conversion hosts enable', 'Create transformation mappings', 'Create transformation plans', 'Start performance monitoring', 'Execute transformation plans', 'Monitor transformation plans']),
        booleanParam(defaultValue: false, description: 'If checked, this will be the ONLY stage to run', name: 'SINGLE_STAGE'),
        choice(defaultValue: '', description: 'Specify the verbosity level of running stages', name: 'VERBOSITY_LEVEL', choices: ['', '-v', '-vv', '-vvv']),
        string(defaultValue: '', description: 'Gerrit refspec for cherry pick', name: 'JENKINS_GERRIT_REFSPEC')
      ]
    ),
  ]
)

def stages_ = other.get_v2v_current_stage(params.START_FROM_STAGE, params.SINGLE_STAGE)

pipeline {
  agent {
    node {
      label params.NODE_LABEL ? params.NODE_LABEL : null
    }
  }
  stages {
    stage ('Main Stage') {
      options {
        lock(resource: "${GE_NAME}")
      }
      stages {
        stage ('Locked Resources') {
          steps {
            script {
              log.info("Locked resources: ${GE_NAME}")
            }
          }
        }
        stage ("Checkout jenkins repository") {
          steps {
            checkout(
              [
                $class: 'GitSCM',
                branches: [[name: 'origin/master']],
                doGenerateSubmoduleConfigurations: false,
                extensions: [
                  [$class: 'RelativeTargetDirectory', relativeTargetDir: 'jenkins'],
                  [$class: 'CleanBeforeCheckout'],
                  [$class: 'CloneOption', depth: 0, noTags: false, reference: '', shallow: true],
                  [$class: 'PruneStaleBranch']
                ],
                submoduleCfg: [],
                userRemoteConfigs: [[url: 'git://git.app.eng.bos.redhat.com/rhevm-jenkins.git']]
              ]
            )
            sh '''echo "Executed from: v2v Jenkinsfile"

            if [ -d $WORKSPACE/jenkins ]
            then
              pushd $WORKSPACE/jenkins
              echo $JENKINS_GERRIT_REFSPEC
              for ref in $JENKINS_GERRIT_REFSPEC ;
              do
                git fetch git://git.app.eng.bos.redhat.com/rhevm-jenkins.git "$ref" && git cherry-pick FETCH_HEAD || (
                    echo \'!!! FAIL TO CHERRYPICK !!!\' "$ref" ; false
                )
              done
              popd
            fi
            '''
          }
        }

        stage ("Generating inventory and extra_vars") {
          steps {
            sh '''
                rm -rf yaml_generator
                virtualenv yaml_generator
                source yaml_generator/bin/activate
                pip install --upgrade pip
                pip install pyyaml jinja2 pathlib
                ${WORKSPACE}/jenkins/tools/v2v/v2v_env.py $SOURCE_YAML \
                                                        --inventory  ${WORKSPACE}/jenkins/qe/v2v/inventory \
                                                        --extra_vars ${WORKSPACE}/extra_vars.yml \
                                                        --trans_method $TRANSPORT_METHODS \
                                                        --image_url $CFME_IMAGE_URL \
                                                        --rhv_hosts "$RHV_HOSTS" \
                                                        --vmw_hosts "$VMW_HOSTS" \
                                                        --number_of_vms $NUMBER_OF_VMS \
                                                        --provider_concurrent_max $PROVIDER_CONCURRENT_MAX \
                                                        --host_concurrent_max $HOST_CONCURRENT_MAX \
                                                        --v2v_ci_vmw_template $VMW_TEMPLATE_NAME \
                                                        --v2v_ci_source_datastore "$VMW_STORAGE_NAME" \
                                                        --v2v_ci_target_datastore "$RHV_STORAGE_NAME" \
                                                        --job_basename_url $JOB_BASE_NAME \
                                                        --rhv_ge "$GE_NAME" \
                                                        --v2v_ci_environment "$ENVIRONMENT" \
                                                        --rhv_storage "$RHV_STORAGE"

                deactivate
                '''
          }
        }

        stage ("oVirt/RHV Recreate Storage Domain") {
          when {
            allOf {
              expression { params.RHV_RECREATE_STORAGE }
              expression { params.ENVIRONMENT == 'RDU'}
            }
          }
          steps {
            v2v_ansible(
              playbook: "miq_run_step.yml",
              extraVars: ['@extra_vars.yml', 'rhv_recreate_storage_domain=true'],
              tags: ['rhv_recreate_storage'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ("ManageIQ/CloudForms Pre-Check Nightly") {
          when {
            expression { params.MIQ_NIGHTLY_PRE_CHECK }
          }
          steps {
            v2v_ansible(
              playbook: "miq_run_step.yml",
              extraVars: ['@extra_vars.yml', 'miq_pre_check_nightly=true'],
              tags: ['miq_pre_check_nightly'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ("ManageIQ/CloudForms Remove existing instance") {
          when {
            expression { params.MIQ_REMOVE_EXISTING_INSTANCE }
          }
          steps {
            v2v_ansible(
              playbook: "miq_run_step.yml",
              extraVars: ['@extra_vars.yml', 'miq_pre_check=true', 'v2v_ci_miq_vm_force_remove=true'],
              tags: ['miq_pre_check'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ("ManageIQ/CloudForms Remove Pre-Check") {
          when {
            expression { params.MIQ_REMOVE_EXISTING_INSTANCE }
          }
          steps {
            v2v_ansible(
              playbook: "miq_run_step.yml",
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_pre_check'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ("Deploy ManageIQ/CloudForms") {
          when {
            expression { params.MIQ_REMOVE_EXISTING_INSTANCE }
          }
          steps {
            v2v_ansible(
              playbook: "miq_deploy.yml",
              extraVars: ['@extra_vars.yml'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Create VMs') {
          when {
            expression { stages_['Create VMs'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_create_vms'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Remove RHV VMs') {
          when {
            expression { stages_['Remove RHV VMs'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['remove_rhv_vms'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Install Nmon') {
          when {
            expression { stages_['Install Nmon'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_install_nmon'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Add extra providers') {
          when {
            expression { stages_['Add extra providers'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_add_extra_providers'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Set RHV provider concurrent VM migration max') {
          when {
            expression { stages_['Set RHV provider concurrent VM migration max'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_set_provider_concurrent_vm_migration_max'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Configure oVirt conversion hosts') {
          when {
            expression { stages_['Configure oVirt conversion hosts'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_config_ovirt_conversion_hosts'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Configure ESX hosts') {
          when {
            expression { stages_['Configure ESX hosts'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_config_vmware_esx_hosts'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('SSH Configuration') {
          when {
            expression { stages_['SSH Configuration'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['ssh_configuration'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Conversion hosts enable') {
          when {
            expression { stages_['Conversion hosts enable'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_conversion_hosts_enable'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Create transformation mappings') {
          when {
            expression { stages_['Create transformation mappings'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_config_infra_mappings'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Create transformation plans') {
          when {
            expression { stages_['Create transformation plans'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_config_migration_plan'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Start performance monitoring') {
          when {
            expression { stages_['Start performance monitoring'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_start_monitoring'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Execute transformation plans') {
          when {
            expression { stages_['Execute transformation plans'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_order_migration_plan'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }

        stage ('Monitor transformation plans') {
          when {
            expression { stages_['Monitor transformation plans'] }
          }
          steps {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_monitor_transformations'],
              verbosity: params.VERBOSITY_LEVEL
            )
          }
        }
      }
      post {
        always {
            v2v_ansible(
              playbook: 'miq_run_step.yml',
              extraVars: ['@extra_vars.yml'],
              tags: ['miq_stop_monitoring'],
              verbosity: params.VERBOSITY_LEVEL
            )
            archiveArtifacts artifacts: 'cfme_logs/*.tar.gz'
            archiveArtifacts artifacts: 'conv_logs/*.tar.gz'
        }
      }
    }
  }
}
