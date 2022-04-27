// Copyright (c) Open Enclave SDK contributors.
// Licensed under the MIT License.

OECI_LIB_VERSION = params.OECI_LIB_VERSION ?: "master"
library "OpenEnclaveJenkinsLibrary@${OECI_LIB_VERSION}"

GLOBAL_TIMEOUT_MINUTES = 240

OETOOLS_REPO_NAME = "oejenkinscidockerregistry.azurecr.io"
OETOOLS_REPO_CREDENTIAL_ID = "oejenkinscidockerregistry"
OETOOLS_DOCKERHUB_REPO_CREDENTIAL_ID = "oeciteamdockerhub"

DOCKER_IMAGES_NAMES = ["oetools-18.04", "oetools-20.04"]
AZURE_IMAGES_MAP = [
    // Mapping between shared gallery image definition name and
    // generated Azure managed image name
    "ubuntu-18.04":    "${params.IMAGE_ID}-ubuntu-18.04-SGX",
    "ubuntu-20.04":    "${params.IMAGE_ID}-ubuntu-20.04-SGX",
    "ws2019-nonSGX":   "${params.IMAGE_ID}-ws2019-nonSGX",
    "ws2019-SGX-DCAP": "${params.IMAGE_ID}-ws2019-SGX-DCAP"
]

// def update_production_docker_images() {
//     node("nonSGX-ubuntu-2004") {
//         timeout(GLOBAL_TIMEOUT_MINUTES) {
//             stage("Backup current production Docker images") {
//                 docker.withRegistry("https://${OETOOLS_REPO_NAME}", OETOOLS_REPO_CREDENTIAL_ID) {
//                     for (image_name in DOCKER_IMAGES_NAMES) {
//                         def image = docker.image("${OETOOLS_REPO_NAME}/${image_name}:latest")
//                         oe.exec_with_retry { image.pull() }
//                         oe.exec_with_retry { image.push("latest-backup") }
//                     }
//                 }
//             }
//         }
//     }
//     node(IMAGES_BUILD_LABEL) {
//         timeout(GLOBAL_TIMEOUT_MINUTES) {
//             stage("Update production Docker images") {
//                 for (image_name in DOCKER_IMAGES_NAMES) {
//                     docker.withRegistry("https://${OETOOLS_REPO_NAME}", OETOOLS_REPO_CREDENTIAL_ID) {
//                         def image = docker.image("${OETOOLS_REPO_NAME}/${image_name}:${env.DOCKER_TAG}")
//                         oe.exec_with_retry { image.pull() }
//                         oe.exec_with_retry { image.push("latest") }
//                     }
//                     sh("docker tag ${OETOOLS_REPO_NAME}/${image_name}:${env.DOCKER_TAG} oeciteam/${image_name}:latest")
//                     docker.withRegistry('', OETOOLS_DOCKERHUB_REPO_CREDENTIAL_ID) {
//                         def image = docker.image("oeciteam/${image_name}:latest")
//                         oe.exec_with_retry { image.push("latest") }
//                     }
//                 }
//             }
//         }
//     }
// }

def update_production_azure_gallery_images(String image_name) {
    timeout(GLOBAL_TIMEOUT_MINUTES) {
        stage("Azure CLI Login") {
            withCredentials([
                    usernamePassword(credentialsId: 'SERVICE_PRINCIPAL_OSTCLAB',
                                        passwordVariable: 'SERVICE_PRINCIPAL_PASSWORD',
                                        usernameVariable: 'SERVICE_PRINCIPAL_ID'),
                    string(credentialsId: 'openenclaveci-subscription-id', variable: 'SUBSCRIPTION_ID'),
                    string(credentialsId: 'TenantID', variable: 'TENANT_ID')]) {
                sh '''#!/bin/bash
                    az login --service-principal -u ${SERVICE_PRINCIPAL_ID} -p ${SERVICE_PRINCIPAL_PASSWORD} --tenant ${TENANT_ID}
                    az account set -s ${SUBSCRIPTION_ID}
                '''
            }
        }
        stage("Update production Azure managed image: ${image_name}") {
            sh """
                IMAGE_ID=\$(az sig image-version show \
                    --resource-group ${params.RESOURCE_GROUP} \
                    --gallery-name "${params.E2E_IMAGES_GALLERY_NAME}" \
                    --gallery-image-definition "${image_name}" \
                    --gallery-image-version ${params.IMAGE_VERSION} \
                    | jq -r '.id')

                az sig image-version delete \
                    --resource-group ${params.RESOURCE_GROUP} \
                    --gallery-name ${params.PRODUCTION_IMAGES_GALLERY_NAME} \
                    --gallery-image-definition ${image_name} \
                    --gallery-image-version ${params.IMAGE_VERSION}

                az sig image-version create \
                    --resource-group ${params.RESOURCE_GROUP} \
                    --gallery-name ${params.PRODUCTION_IMAGES_GALLERY_NAME} \
                    --gallery-image-definition ${image_name} \
                    --gallery-image-version ${params.IMAGE_VERSION} \
                    --managed-image \${IMAGE_ID} \
                    --target-regions ${params.REPLICATION_REGIONS.split(',').join(' ')} \
                    --replica-count 1
            """
        }
    }
}

// def parallel_steps = [ "Update Docker images": { update_production_docker_images() } ]
def parallel_steps = [:]
AZURE_IMAGES_MAP.keySet().each {
    image_name -> parallel_steps["Update Azure gallery ${image_name} image"] = { update_production_azure_gallery_images(image_name) }
}

pipeline {
    agent {
        label globalvars.AGENTS_LABELS["vanilla-ubuntu-2004"]
    }
    options {
        timeout(time: 240, unit: 'MINUTES')
    }
    parameters {
        string(name: 'REPOSITORY_NAME', defaultValue: 'openenclave/openenclave', description: '[OPTIONAL] GitHub repository to checkout')
        string(name: 'BRANCH_NAME', defaultValue: 'master', description: '[OPTIONAL] The branch used to checkout the repository')
        string(name: 'RESOURCE_GROUP', defaultValue: 'jenkins-images', description: '[OPTIONAL] The resouce group containing the Azure images')
        string(name: 'IMAGE_ID', description: '[REQUIRED] The image id to promote from E2E to Production. E.g. 2021.12.0336')
        string(name: 'E2E_IMAGES_GALLERY_NAME', defaultValue: 'e2e_images', description: '[OPTIONAL] The Azure Shared Image Gallery for E2E Images')
        string(name: 'PRODUCTION_IMAGES_GALLERY_NAME', defaultValue: 'production_images', description: '[OPTIONAL] The Azure Shared Image Gallery for Production Images')
        string(name: 'IMAGE_VERSION', defaultValue: '${IMAGE_ID}', description: '[OPTIONAL] The version that the image should be tagged as')
        string(name: 'REPLICATION_REGIONS', defaultValue: 'westus,westeurope,eastus,uksouth,eastus2,canadacentral', description: '[OPTIONAL] Replication regions for the shared gallery images definitions (comma-separated)')
        string(name: "OECI_LIB_VERSION", defaultValue: 'master', description: '[OPTIONAL] Version of OE Libraries to use')
    }
    stages {
        stage("Install Azure CLI") {
            steps {
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
                        sudo apt-get -y install azure-cli jq
                    """
                }
            }
        }
        stage("Promote images") {
            steps {
                script {
                    parallel parallel_steps
                }
            }
        }
    }
    post {
        always {
            sh """
                az logout || true
                az cache purge
                az account clear
            """
        }
    }
}
