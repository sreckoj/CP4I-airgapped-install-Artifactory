
# Air-gapped installation of IBM CP4I using the Artifactory registry


## Command line tools

  - We need
    - docker or podman
    - OpenShift CLI (oc)
    - IBM oc plugin called ibm-pak

  - Download ibm-pak using podman:
    ```
    id=$(podman create cp.icr.io/cpopen/cpfs/ibm-pak:v1.3.0 - )
    podman cp $id:/ibm-pak-plugin plugin-dir
    podman rm -v $id
    cd plugin-dir
    ```

  - Or, using docker:
    ```
    id=$(docker create cp.icr.io/cpopen/cpfs/ibm-pak:v1.3.0 - )
    docker cp $id:/ibm-pak-plugin plugin-dir
    docker rm -v $id
    cd plugin-dir
    ```

  - Extract the plugin (in our case for x86 Linux) and copy it to the executable path:
    ```
    tar -xf oc-ibm_pak-linux-amd64.tar.gz
    chmod 755 oc-ibm_pak-linux-amd64
    mv oc-ibm_pak-linux-amd64 /usr/local/bin/oc-ibm_pak
    ```

  - Verify:
    ```
    oc ibm-pak --help
    ```  

## Download CASE files

  - The following table shows the case names for all components of the Cloud Pak for Integration

    | Component                    | CASE name                              |
    |------------------------------|----------------------------------------|
    | Platform Navigator           | ibm-integration-platform-navigator     |
    | Automation foundation assets | ibm-integration-asset-repository       |
    | Operations Dashboard         | ibm-integration-operations-dashboard   |
    | API Connnect                 | ibm-apiconnect                         |
    | App Connect                  | ibm-appconnect                         |
    | MQ                           | ibm-mq                                 |
    | Event Streams                | ibm-eventstreams                       |
    | DataPower Gateway            | ibm-datapower-operator                 |
    | Aspera HSTS                  | ibm-aspera-hsts-operator               |

  - To obtain the lastest version append the CASE name to the following URL:
    https://github.com/IBM/cloud-pak/tree/master/repo/case

  - For example, this shows all versions of the **Platform Navigator**:
    https://github.com/IBM/cloud-pak/tree/master/repo/case/ibm-integration-platform-navigator
    At the moment of writing this, the latest was **1.7.3** 

  - And this shows all versions of the **Event Streams**:
    https://github.com/IBM/cloud-pak/tree/master/repo/case/ibm-eventstreams
    At the moment of writing this, the latest was **1.6.6**

  > Note: The CASE version is not same as the product version. For example, the Event Streams CASE verson 1.6.6 contains IBM Event Streams 11.0.4. The same repository contains also *index.yaml* file with the CASE version - product version mappings.

 
  - In this example we will install only the **Platform Navigator** and the **Event Streams**, 
    repeat the same steps for other components. We will define environment variables with the CASE names and versions:
    ```
    export PN_CASE_NAME=ibm-integration-platform-navigator
    export PN_CASE_VERSION=1.7.3
    export ES_CASE_NAME=ibm-eventstreams
    export ES_CASE_VERSION=1.6.6
    ```

  - Run commands to download files 
    (note: despite it is using `oc` CLI you don't need to login to OpenShift). The files will be stored in the directory `~/.ibm-pak/`
    ```
    oc ibm-pak get $PN_CASE_NAME --version $PN_CASE_VERSION
    oc ibm-pak get $ES_CASE_NAME --version $ES_CASE_VERSION
    ```

## Generate mirros manifests

  - Define environment variable with the target registry host and port 
    (seems that in case of the Artifactory, we can skip the port)
    ```
    export TARGET_REGISTRY=sjdemo.jfrog.io
    ```

  - Generate manifests (they will be created in `~/.ibm-pak/data/mirror/`)
    ```
    oc ibm-pak generate mirror-manifests $PN_CASE_NAME --version $PN_CASE_VERSION $TARGET_REGISTRY
    oc ibm-pak generate mirror-manifests $ES_CASE_NAME --version $ES_CASE_VERSION $TARGET_REGISTRY
    ```

## Authenticate to the registries

  - In case of using docker the login credentals are stored in `$HOME/.docker/config.json`
    In case of using podman they are stored in the same file or in `${XDG_RUNTIME_DIR}/containers/auth.json`. Verify which of those files are used and create an environment variable with path to it:
    ```
    export REGISTRY_AUTH_FILE=$HOME/.docker/config.json
    ```  
    or
    ```
    export REGISTRY_AUTH_FILE=${XDG_RUNTIME_DIR}/containers/auth.json
    ```

  - Obtain the key for the IBM entitled registry:
    https://myibm.ibm.com/products-services/containerlibrary

  - Create the environment variable with the key
    ```
    export ENTITLEMENT_KEY=...
    ```

  - Login to the entitled registry:
    ```
    docker login cp.icr.io --username cp --password $ENTITLEMENT_KEY
    ```
    or
    ```
    podman login cp.icr.io --username cp --password $ENTITLEMENT_KEY
    ```

  - Authenticate to the local registry
    ```
    docker login $TARGET_REGISTRY
    ```
    or
    ```
    podman login $TARGET_REGISTRY
    ```
    Enter username and password.

  - Verify that the credentials for both registries are stored in the file
    ```
    cat $REGISTRY_AUTH_FILE
    ```  

## Create Artifactory repositories

  - In is not clear if this is specific for the particular Artifactory instance or 
    it is true in general. Before mirroring we have to manually create repositories. Check the content of `images-mapping.txt` file for all (in our case only two) components:
    ```
    cat ~/.ibm-pak/data/mirror/$PN_CASE_NAME/$PN_CASE_VERSION/images-mapping.txt
    cat ~/.ibm-pak/data/mirror/$ES_CASE_NAME/$ES_CASE_VERSION/images-mapping.txt
    ```
    As an example, for image `cp.icr.io/cp/icp4i/icip-navigator...` you will have to create repository `cp`, or for image `icr.io/cpopen/cpfs/zen-core-api...` you have to create repository `cpopen` etc... 

  - Use the following steps to create a local repository in Artifactory
    - Select **Repositories**
    - Click **Add Repositories/Local Repository**
    - Select **Docker** type
    - Under **Basic** settings
      - Enter the repository name in: **Repository Key**
      - Be sure that *V2* is selected for the **API version**
    - Click on **Create Repository** 

  - The following repositories have to be created for the **Platform Navigator**:
    ```
    cp
    cpopen
    opencloudio
    ```

  - And the following for the **Event Streams** (all of them, except ibmcom are the same as for PN):
    ```
    cp
    ibmcom
    cpopen
    opencloudio
    ```

## Mirror images

  > Note: If the certificate of the local registry is not trusted add `--insecure` to the bellow commands

  - Platform Navigator
    ```
    oc image mirror -f ~/.ibm-pak/data/mirror/$PN_CASE_NAME/$PN_CASE_VERSION/images-mapping.txt -a $REGISTRY_AUTH_FILE --filter-by-os '.*' --skip-multiple-scopes --max-per-registry=1
    ```

  - Event Streams  
    ```
    oc image mirror -f ~/.ibm-pak/data/mirror/$ES_CASE_NAME/$ES_CASE_VERSION/images-mapping.txt -a $REGISTRY_AUTH_FILE --filter-by-os '.*' --skip-multiple-scopes --max-per-registry=1
    ```

## Configure cluster

  - Login to the OpenShift cluster

  - Download global pull secret
    ```
    oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' > global-pull-secret.json
    ```

  - Append local registry credentials to secret file
    ```
    oc registry login --registry="sjdemo.jfrog.io" --auth-basic="srecko.janjic@si.ibm.com:********" --to=global-pull-secret.json
    ```

  - Updat global pull secret
    ```
    oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=global-pull-secret.json
    ```

  > You can update the secret also using the OpenShift web console. Navigate to the project **openshift-config** and edit secret **pull-secret**. Add registry address, username and password  

  - Create image content source policy
    ```
    oc apply -f  ~/.ibm-pak/data/mirror/$PN_CASE_NAME/$PN_CASE_VERSION/image-content-source-policy.yaml

    oc apply -f  ~/.ibm-pak/data/mirror/$ES_CASE_NAME/$ES_CASE_VERSION/image-content-source-policy.yaml
    ```

  - Verify. Two content source policies should be created in our case
    ```
    oc get imageContentSourcePolicy
    ```
    Result
    ```
    NAME                                 AGE
    ibm-eventstreams                     24s
    ibm-integration-platform-navigator   33s        
    ```

  - Wait that all nodes are updated
    ```
    oc get MachineConfigPool -w
    ```
    Result
    ```
    NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED
    master   rendered-master-21916e49162c2b47604e1a7cdd655df6   True      False      False   
    worker   rendered-worker-0bb8843c00c46218515165f18979aedd   True      False      False   
    ```
    You can also check with
    ```
    oc get nodes
    ```
    None of them should show `SchedulingDisabled` before you proceed

  - (Optional)
    If your local registry uses a self-signed certificate you can add it to the insecure regostries list
    ```
    oc patch image.config.openshift.io/cluster --type=merge -p '{"spec":{"registrySources":{"insecureRegistries":["'${TARGET_REGISTRY}'"]}}}'
    ``` 

## Add catalog sources

  - When using CASE files we need to add the catalog source for each product separatelly
    ```
    oc apply -f ~/.ibm-pak/data/mirror/${PN_CASE_NAME}/${PN_CASE_VERSION}/catalog-sources.yaml
    oc apply -f ~/.ibm-pak/data/mirror/${ES_CASE_NAME}/${ES_CASE_VERSION}/catalog-sources.yaml
    ```

  - Verify
    ```
    oc get catalogsource -n openshift-marketplace
    ```

## Install operators using CLI

  - (Optional)
    If operators are installed in specific namespace the operator group must be created.
    Replace `<namespace>` with the namespace (project) in which operator will be installed and apply the yaml:
    ```
    cat << EOF | oc apply -f -
    ---
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: ibm-integration-operatorgroup
      namespace: <namespace>
    spec:
      targetNamespaces:
      - <namespace>    
    EOF
    ```

  > Note: The following examples show installation in all namespaces and with the latest channel.
    If installing in a specific namespace, replace the `openshift-operators` with that namespace.

  - Create Platform Navigator subscription
    ```
    cat << EOF | oc apply -f -
    ---
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: ibm-integration-platform-navigator
      namespace: openshift-operators
    spec:
      channel: v6.0
      installPlanApproval: Manual
      name: ibm-integration-platform-navigator
      source: ibm-integration-platform-navigator-catalog
      sourceNamespace: openshift-marketplace    
    EOF
    ```

  - Check if there are install plans that have to be approved
    ```
    oc get installplan --all-namespaces
    ```  
    Sample result:
    ```
    NAMESPACE             NAME            CSV                                   APPROVAL   APPROVED
    openshift-operators   install-cl5jr   ibm-common-service-operator.v3.19.4   Manual     false    
    ```

  - Approve listed install plans
    ```
    oc patch installplan install-mnwz2 --namespace openshift-operators --type merge --patch '{"spec":{"approved":true}}'
    ```

  - Create Event Streams subscription
    ```
    cat << EOF | oc apply -f -
    ---
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: ibm-eventstreams
      namespace: openshift-operators
    spec:
      channel: v3.0
      installPlanApproval: Manual
      name: ibm-eventstreams
      source: ibm-eventstreams
      sourceNamespace: openshift-marketplace    
    EOF
    ```

  - Repeat approval steps as explained above.  


  - Verify operators
    ```
    oc get csv -n openshift-operators
    ```


## Install capabilities instances using CLI

  - Create namespace (project)
    ```
    oc new-project cp4i
    ```

  - Create secret with the entitlement key 
    ```
    export ENTITLEMENT_KEY=...

    oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=$ENTITLEMENT_KEY --docker-server=cp.icr.io --namespace=cp4i
    ```

  - Create instance of Platform Navigator
    **Note:** Accept license and correct the storage class before running.
    ```
    cat << EOF | oc apply -f -
    ---    
    apiVersion: integration.ibm.com/v1beta1
    kind: PlatformNavigator
    metadata:
      name: integration-navigator
      namespace: cp4i
    spec:
      license:
        accept: false
        license: L-RJON-CD3JKX
      storage:
        class: rook-cephfs
      mqDashboard: true
      replicas: 1
      version: 2022.2.1
    EOF
    ```

  - Check if there are operators that are waiting for approval 
    (they can appear in ibm-common-services project)  
     **Repeat this step several times**
    ```
    oc get installplan --all-namespaces

    oc patch installplan install-b98sk --namespace ibm-common-services --type merge --patch '{"spec":{"approved":true}}'
    ```

  - Check the status of the Platform Navigator custom resource
    ```
    oc get PlatformNavigator -n cp4i
    ```
    Wait until it is ready
    To see more details use command:
    ```
    oc describe PlatformNavigator integration-navigator -n cp4i
    ```


