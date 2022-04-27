// Copyright (c) Open Enclave SDK contributors.
// Licensed under the MIT License.

import java.time.*
import java.time.format.DateTimeFormatter

OECI_LIB_VERSION = env.OECI_LIB_VERSION ?: "master"
library "OpenEnclaveJenkinsLibrary@${params.OECI_LIB_VERSION}"

GLOBAL_TIMEOUT_MINUTES = 480

JENKINS_USER_CREDS_ID = 'oeadmin-credentials'
OETOOLS_REPO = 'oejenkinscidockerregistry.azurecr.io'
OETOOLS_REPO_CREDENTIALS_ID = 'oejenkinscidockerregistry'
SERVICE_PRINCIPAL_CREDENTIALS_ID = 'SERVICE_PRINCIPAL_OSTCLAB'
AZURE_IMAGES_MAP = [
    "win2019": [
        "image": "MicrosoftWindowsServer:WindowsServer:2019-datacenter-gensecond:latest",
        "generation": "V2"
    ]
]
OS_NAME_MAP = [
    "win2019": "Windows Server 2019",
    "ubuntu":  "Ubuntu",
]

def get_image_version() {
    if (params.IMAGE_VERSION) {
        return "${params.IMAGE_VERSION}"
    }
    def now = LocalDateTime.now()
    return (now.format(DateTimeFormatter.ofPattern("yyyy")) + "." + \
            now.format(DateTimeFormatter.ofPattern("MM")) + "." + \
            now.format(DateTimeFormatter.ofPattern("dd")) + "${BUILD_NUMBER}")
}

def get_commit_id() {
    def last_commit_id = sh(script: "git rev-parse --short HEAD", returnStdout: true).tokenize().last()
    return last_commit_id
}

def buildLinuxManagedImage(String os_type, String version, String managed_image_name_id, String gallery_image_version) {
    stage('Check Prerequisites') {
        retry(10) {
            sh """#!/bin/bash
                sleep 5
                curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
                sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com \$(lsb_release -cs) main"
                ${helpers.WaitForAptLock()}
                sudo apt-get update && sudo apt-get install packer
            """
        }
    }
    stage("${OS_NAME_MAP[os_type]} ${version} Build") {
        withEnv([
                "DOCKER_REGISTRY=${OETOOLS_REPO}",
                "MANAGED_IMAGE_NAME_ID=${managed_image_name_id}",
                "GALLERY_IMAGE_VERSION=${gallery_image_version}",
                "RESOURCE_GROUP=${params.RESOURCE_GROUP}",
                "GALLERY_NAME=${params.GALLERY_NAME}"]) {
            stage("Run Packer Job") {
                timeout(GLOBAL_TIMEOUT_MINUTES) {
                    withCredentials([
                            usernamePassword(credentialsId: JENKINS_USER_CREDS_ID,
                                             usernameVariable: "SSH_USERNAME",
                                             passwordVariable: "SSH_PASSWORD"),
                            usernamePassword(credentialsId: OETOOLS_REPO_CREDENTIALS_ID,
                                             usernameVariable: "DOCKER_USER_NAME",
                                             passwordVariable: "DOCKER_USER_PASSWORD"),
                            usernamePassword(credentialsId: SERVICE_PRINCIPAL_CREDENTIALS_ID,
                                             passwordVariable: 'SERVICE_PRINCIPAL_PASSWORD',
                                             usernameVariable: 'SERVICE_PRINCIPAL_ID'),
                            string(credentialsId: 'OSCTLabSubID', variable: 'SUBSCRIPTION_ID'),
                            string(credentialsId: 'TenantID', variable: 'TENANT_ID')]) {
                        sh '''#!/bin/bash
                            az login --service-principal -u ${SERVICE_PRINCIPAL_ID} -p ${SERVICE_PRINCIPAL_PASSWORD} --tenant ${TENANT_ID}
                            az account set -s ${SUBSCRIPTION_ID}
                        '''
                        retry(5) {
                            sh """#!/bin/bash
                                packer build -force \
                                    -var-file=${WORKSPACE}/.jenkins/infrastructure/provision/templates/packer/azure_managed_image/${os_type}-${version}-variables.json \
                                    -var "use_azure_cli_auth=true" \
                                    ${WORKSPACE}/.jenkins/infrastructure/provision/templates/packer/azure_managed_image/packer-${os_type}.json
                            """
                        }
                    }
                }
            }
        }
    }
}

def buildWindowsManagedImage(String os_series, String img_name_suffix, String launch_configuration, String image_id, String image_version) {

    stage("${launch_configuration} Build") {
        withCredentials([usernamePassword(credentialsId: JENKINS_USER_CREDS_ID,
                                    usernameVariable: "JENKINS_USER_NAME",
                                    passwordVariable: "JENKINS_USER_PASSWORD")]) {
            def az_login_script = """
                az login --service-principal -u \$SERVICE_PRINCIPAL_ID -p \$SERVICE_PRINCIPAL_PASSWORD --tenant \$TENANT_ID
                az account set -s \$SUBSCRIPTION_ID
            """
            def managed_image_name_id = image_id
            def gallery_image_version = image_version
            def vm_rg_name = "build-${managed_image_name_id}-${img_name_suffix}-${BUILD_NUMBER}"
            def vm_name = "${os_series}-vm"
            def jenkins_rg_name = params.JENKINS_RESOURCE_GROUP
            def jenkins_vnet_name = params.JENKINS_VNET_NAME
            def jenkins_subnet_name = params.JENKINS_SUBNET_NAME
            def azure_image_id = AZURE_IMAGES_MAP[os_series]["image"]

            stage("Debug") {
                sh """
                    whoami
                    id
                    newgrp docker
                    id
                    stat /var/run/docker.sock
                """
            }
            stage("Prepare Resource Group") {
                def az_rg_create_script = """
                    ${az_login_script}
                    az group create --name ${vm_rg_name} --location ${REGION}
                """
                common.azureEnvironment(az_rg_create_script, params.OE_DEPLOY_IMAGE)
            }

            try {
                stage("Provision VM") {
                    def provision_script = """
                        ${az_login_script}

                        SUBNET_ID=\$(az network vnet subnet show \
                            --resource-group ${jenkins_rg_name} \
                            --name ${jenkins_subnet_name} \
                            --vnet-name ${jenkins_vnet_name} --query id -o tsv)

                        az vm create \
                            --resource-group ${vm_rg_name} \
                            --location ${REGION} \
                            --name ${vm_name} \
                            --size Standard_DC4s \
                            --os-disk-size-gb 128 \
                            --subnet \$SUBNET_ID \
                            --admin-username ${JENKINS_USER_NAME} \
                            --admin-password ${JENKINS_USER_PASSWORD} \
                            --image ${azure_image_id}
                    """
                    common.azureEnvironment(provision_script, params.OE_DEPLOY_IMAGE)
                }

                stage("Deploy VM") {
                    def deploy_script = """
                        ${az_login_script}

                        VM_DETAILS=\$(az vm show --resource-group ${vm_rg_name} \
                                                --name ${vm_name} \
                                                --show-details)

                        az vm run-command invoke \
                            --resource-group ${vm_rg_name} \
                            --name ${vm_name} \
                            --command-id EnableRemotePS

                        PRIVATE_IP=\$(echo \$VM_DETAILS | jq -r '.privateIps')
                        rm -f ${WORKSPACE}/scripts/ansible/inventory/hosts-${launch_configuration}
                        echo "[windows-agents]" >> ${WORKSPACE}/scripts/ansible/inventory/hosts-${launch_configuration}
                        echo "\$PRIVATE_IP" >> ${WORKSPACE}/scripts/ansible/inventory/hosts-${launch_configuration}
                        echo "ansible_user: ${JENKINS_USER_NAME}" >> ${WORKSPACE}/scripts/ansible/inventory/host_vars/\$PRIVATE_IP
                        echo "ansible_password: ${JENKINS_USER_PASSWORD}" >> ${WORKSPACE}/scripts/ansible/inventory/host_vars/\$PRIVATE_IP
                        echo "ansible_winrm_transport: ntlm" >> ${WORKSPACE}/scripts/ansible/inventory/host_vars/\$PRIVATE_IP
                        echo "launch_configuration: ${launch_configuration}" >> ${WORKSPACE}/scripts/ansible/inventory/host_vars/\$PRIVATE_IP

                        cd ${WORKSPACE}/scripts/ansible
                        source ${WORKSPACE}/.jenkins/infrastructure/provision/utils.sh
                        ansible-playbook -i ${WORKSPACE}/scripts/ansible/inventory/hosts-${launch_configuration} oe-windows-acc-setup.yml jenkins-packer.yml

                        az vm run-command invoke \
                            --resource-group ${vm_rg_name} \
                            --name ${vm_name} \
                            --command-id RunPowerShellScript \
                            --scripts @${WORKSPACE}/.jenkins/infrastructure/provision/run-sysprep.ps1
                    """
                    common.exec_with_retry(10, 30) {
                        common.azureEnvironment(deploy_script, params.OE_DEPLOY_IMAGE)
                    }
                }


                stage("Generalize VM") {
                    timeout(GLOBAL_TIMEOUT_MINUTES) {
                        def generalize_script = """
                            ${az_login_script}

                            az vm deallocate --resource-group ${vm_rg_name} --name ${vm_name}
                            az vm generalize --resource-group ${vm_rg_name} --name ${vm_name}
                        """
                        common.exec_with_retry(10, 30) {
                            common.azureEnvironment(generalize_script, params.OE_DEPLOY_IMAGE)
                        }
                    }
                }

                stage("Capture Image") {
                    timeout(GLOBAL_TIMEOUT_MINUTES) {
                        def capture_script = """
                            ${az_login_script}

                            VM_ID=\$(az vm show \
                                --resource-group ${vm_rg_name} \
                                --name ${vm_name} | jq -r '.id' )

                            # If the target image doesn't exist, the below command
                            # will not fail because it is idempotent.
                            az image delete \
                                --resource-group ${RESOURCE_GROUP} \
                                --name ${managed_image_name_id}-${img_name_suffix}

                            az image create \
                                --resource-group ${RESOURCE_GROUP} \
                                --name ${managed_image_name_id}-${img_name_suffix} \
                                --hyper-v-generation ${AZURE_IMAGES_MAP[os_series]["generation"]} \
                                --source \$VM_ID
                            """
                        common.exec_with_retry(10, 30) {
                            common.azureEnvironment(capture_script, params.OE_DEPLOY_IMAGE)
                        }
                    }
                }

                stage("Upload Image") {
                    timeout(GLOBAL_TIMEOUT_MINUTES) {
                        def upload_script = """
                            ${az_login_script}

                            MANAGED_IMG_ID=\$(az image show \
                                --resource-group ${RESOURCE_GROUP} \
                                --name ${managed_image_name_id}-${img_name_suffix} \
                                | jq -r '.id' )

                            # If the target image version doesn't exist, the below
                            # command will not fail because it is idempotent.
                            az sig image-version delete \
                                --resource-group ${RESOURCE_GROUP} \
                                --gallery-name ${GALLERY_NAME} \
                                --gallery-image-definition ${img_name_suffix} \
                                --gallery-image-version ${gallery_image_version}

                            az sig image-version create \
                                --resource-group ${RESOURCE_GROUP} \
                                --gallery-name ${GALLERY_NAME} \
                                --gallery-image-definition ${img_name_suffix} \
                                --gallery-image-version ${gallery_image_version} \
                                --managed-image \$MANAGED_IMG_ID \
                                --target-regions ${env.REPLICATION_REGIONS.split(',').join(' ')} \
                                --replica-count 1
                        """
                        common.exec_with_retry(10, 30) {
                            common.azureEnvironment(upload_script, params.OE_DEPLOY_IMAGE)
                        }
                    }
                }

            } finally {
                stage("${img_name_suffix}-cleanup") {
                    def az_rg_cleanup_script = """
                        ${az_login_script}
                        az group delete --name ${vm_rg_name} --yes
                        az image delete \
                            --resource-group ${RESOURCE_GROUP} \
                            --name ${managed_image_name_id}-${img_name_suffix}
                    """
                    common.azureEnvironment(az_rg_cleanup_script, params.OE_DEPLOY_IMAGE)
                }
            }
        }
    }
}

node(params.AGENTS_LABEL) {
    stage("Initialize Workspace") {

        cleanWs()
        checkout scm

        commit_id = get_commit_id()
        version = get_image_version()

        image_version = params.IMAGE_VERSION ?: version
        image_id = params.IMAGE_ID ?: "${version}-${commit_id}"

        println("IMAGE_VERSION: ${image_version}\nIMAGE_ID: ${image_id}")
    }
    stage("Install Azure CLI") {
        retry(10) {
            sh """
                sleep 5
                ${helpers.WaitForAptLock()}
                sudo apt-get update
                sudo apt-get -y install ca-certificates curl apt-transport-https lsb-release gnupg
                curl -sL https://packages.microsoft.com/keys/microsoft.asc |
                    gpg --dearmor |
                    sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
                AZ_REPO=\$(lsb_release -cs)
                echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ \$AZ_REPO main" |
                    sudo tee /etc/apt/sources.list.d/azure-cli.list
                ${helpers.WaitForAptLock()}
                sudo apt-get update
                sudo apt-get -y install azure-cli
            """
        }
    }
    stage("Install Ansible") {
        retry(10) {
            sh """#!/bin/bash
                ${helpers.WaitForAptLock()}
                sudo ${WORKSPACE}/scripts/ansible/install-ansible.sh
            """
        }
    }
    stage("Install Docker") {
        retry(10) {
            sh """#!/bin/bash
                ${helpers.WaitForAptLock()}
                sudo apt-get update
                sudo apt-get install -y \
                    ca-certificates \
                    curl \
                    gnupg \
                    lsb-release
                curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
                echo \
                    "deb [arch=\$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
                    \$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
                sudo apt-get update
                sudo apt-get install -y docker-ce docker-ce-cli containerd.io
                sudo usermod -aG docker \$(whoami)
                sudo cat /etc/group
                newgrp docker
            """
        }
    }
    stage("Build Agents") {
        parallel "Build Windows Server 2019 - nonSGX"       : { buildWindowsManagedImage("win2019", "ws2019-nonSGX", "SGX1FLC-NoIntelDrivers", image_id, image_version) },
                 "Build Windows Server 2019 - SGX1"         : { buildWindowsManagedImage("win2019", "ws2019-SGX", "SGX1", image_id, image_version) },
                 "Build Windows Server 2019 - SGX1FLC DCAP" : { buildWindowsManagedImage("win2019", "ws2019-SGX-DCAP", "SGX1FLC", image_id, image_version) }
                //  "Build Ubuntu 18.04" :                       { buildLinuxManagedImage("ubuntu", "18.04", image_id, image_version) },
                //  "Build Ubuntu 20.04" :                       { buildLinuxManagedImage("ubuntu", "20.04", image_id, image_version) }
    }
    stage("Clean Workspace") {
        cleanWs()
    }
}
