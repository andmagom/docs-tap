# Upgrading Tanzu Application Platform

This document describes how to upgrade Tanzu Application Platform.

You can perform a fresh install of Tanzu Application Platform by following the instructions in [Installing Tanzu Application Platform](install-intro.md).

## <a id='prereqs'></a> Prerequisites

Before you upgrade Tanzu Application Platform:

- Verify that you meet all the [prerequisites](prerequisites.md) of the target Tanzu Application Platform version. If the target Tanzu Application Platform version does not support your existing Kubernetes version, VMware recommends upgrading to a supported version before proceeding with the upgrade.
- For information about installing your Tanzu Application Platform, see [Install your Tanzu Application Platform profile](install.md#install-profile).
- Ensure that Tanzu CLI is updated to the version recommended by the target Tanzu Application Platform version. For information about installing or updating the Tanzu CLI and plug-ins, see [Install or update the Tanzu CLI and plug-ins](install-tanzu-cli.hbs.md#cli-and-plugin).
- For information about Tanzu Application Platform GUI considerations, see [Tanzu Application Platform GUI Considerations](tap-gui/upgrades.md#considerations).
- Verify all packages are reconciled by running `tanzu package installed list -A`.
- To avoid the temporary warning state that is described in [Update the new package repository](#add-new-package-repo), upgrade to Cluster Essentials v{{ vars.url_version }}. See [Cluster Essentials documentation](https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/{{ vars.url_version }}/cluster-essentials/GUID-deploy.html#upgrade) for more information about the upgrade procedures.

### <a id="upgrade-workaround"></a>Upgrade workaround 

>**Note:** If you attempt to upgrade from Tanzu Application Platform v1.2, proceed to the next section [Update the new package repository](#add-new-package-repo).

Follow these workaround steps only when you attempt to upgrade from Tanzu Application Platform v1.1 or below:

1. Install the kp cli by using one of the following sources:  

    - [Open Source](https://github.com/vmware-tanzu/kpack-cli/releases)
    - [VMware Tanzu Network](https://network.pivotal.io/products/build-service/) 

    >**Note:** VMware Tanzu Network requires accepting an additional EULA for the Tanzu Build Service, which is part of Tanzu Application Platform.

    For more information, see [kpack-cli documentation](https://github.com/vmware-tanzu/kpack-cli) in GitHub.

1. Docker log in to the Tanzu Network registry and the installation registry:

    ```console
    docker login registry.tanzu.vmware.com
    docker login $INSTALL_REGISTRY_HOSTNAME
    ```

    Where `INSTALL_REGISTRY_HOSTNAME` is the environment variable you defined earlier while installation. For more information, see [Relocate images to a registry](install.html#relocate-images-to-a-registry).

1. Run the following command with the `kp` cli:
    
    ```console
    kp lifecycle patch -i registry.tanzu.vmware.com/tanzu-application-platform/tap-packages@sha256:851d49d1a9786d6b3fa17fa5d669dbbb687e23d047e6c08165d629603f0d70ec
    ```
    
1. Proceed to the next section [Update the new package repository](#add-new-package-repo).

## <a id="add-new-package-repo"></a> Update the new package repository

Follow these steps to update the new package repository:

1. Relocate the latest version of Tanzu Application Platform images by following step 1 through step 4 in [Relocate images to a registry](install.html#add-tap-package-repo).

    >**Note:** Make sure to update the `TAP_VERSION` to the target version of Tanzu Application Platform you are migrating to. For example, `{{ vars.tap_version }}`. 

1. Add the target version of the Tanzu Application Platform package repository:

    - If you are using Cluster Essentials 1.2 or above, run:

        ```console
        tanzu package repository add NEW-TANZU-TAP-REPOSITORY \
        --url ${INSTALL_REGISTRY_HOSTNAME}/TARGET-REPOSITORY/tap-packages:${TAP_VERSION} \
        --namespace tap-install
        ```

        Where `NEW-TANZU-TAP-REPOSITORY` is the new repository name. Do not use the existing Tanzu package repostitory name with the previous version of Tanzu Application Platform packages.

    - If you are using Cluster Essentials 1.1 or 1.0, run:

        ```console
       tanzu package repository update tanzu-tap-repository \
        --url ${INSTALL_REGISTRY_HOSTNAME}/TARGET-REPOSITORY/tap-packages:${TAP_VERSION} \
        --namespace tap-install
        ```

        >**Note:** If you are using Cluster Essentials 1.0 or 1.1, expect to see the installed Tanzu Application Platform packages in a temporary “Reconcile Failed” state, following a “Package not found” warning. These warnings will disappear after you upgrade the installed Tanzu Application Platform packages to version 1.2.0. 

1. Verify you have added the new package repository by running:

    ```console
    tanzu package repository get tanzu-tap-repository --namespace tap-install
    ```

## <a id="upgrade-tap"></a> Perform the upgrade of Tanzu Application Platform

### <a id="profile-based-instruct"></a> Upgrade instructions for Profile-based installation

For Tanzu Application Platform that is installed by profile, you can perform the upgrade by running:

>**Note:** Ensure you run the following command in the directory where the `tap-values.yaml` file resides.

```console
tanzu package installed update tap -p tap.tanzu.vmware.com -v ${TAP_VERSION}  --values-file tap-values.yaml -n tap-install
```

>**Note:** When upgrading to Tanzu Application Platform v1.2, Tanzu Build Service image resources automatically
>run a build that fails due to a missing dependency.
>This error does not persist and any subsequent builds will resolve this error.
>You can safely wait for the next build of the workloads, which is triggered by new source code changes.
>If you do not want to wait for subsequent builds to run automatically,
>follow the instructions in the troubleshooting item
>[Builds fail after upgrading to Tanzu Application Platform v1.2](tanzu-build-service/troubleshooting.md#tbs-1-2-breaking-change).

### <a id="comp-specific-instruct"></a> Upgrade instructions for component-specific installation

For information about upgrading Tanzu Application Platform GUI, see [upgrading Tanzu Application Platform GUI](tap-gui/upgrades.html).
For information about upgrading Supply Chain Security Tools - Scan, see [Upgrading Supply Chain Security Tools - Scan](scst-scan/upgrading.md).

## <a id="verify"></a> Verify the upgrade

Verify the versions of packages after the upgrade by running:

```console
tanzu package installed list --namespace tap-install
```

Your output is similar, but probably not identical, to the following example output:

```console
- Retrieving installed packages...
  NAME                                PACKAGE-NAME                                         PACKAGE-VERSION  STATUS               
  accelerator                         accelerator.apps.tanzu.vmware.com                    1.3.0            Reconcile succeeded  
  api-auto-registration               apis.apps.tanzu.vmware.com                           0.1.1            Reconcile succeeded  
  api-portal                          api-portal.tanzu.vmware.com                          1.2.2            Reconcile succeeded  
  appliveview                         backend.appliveview.tanzu.vmware.com                 1.3.0            Reconcile succeeded  
  appliveview-connector               connector.appliveview.tanzu.vmware.com               1.3.0            Reconcile succeeded  
  appliveview-conventions             conventions.appliveview.tanzu.vmware.com             1.3.0            Reconcile succeeded  
  appsso                              sso.apps.tanzu.vmware.com                            2.0.0            Reconcile succeeded  
  buildservice                        buildservice.tanzu.vmware.com                        1.7.1            Reconcile succeeded  
  cartographer                        cartographer.tanzu.vmware.com                        0.5.3            Reconcile succeeded  
  cert-manager                        cert-manager.tanzu.vmware.com                        1.7.2+tap.1      Reconcile succeeded  
  cnrs                                cnrs.tanzu.vmware.com                                2.0.1            Reconcile succeeded  
  contour                             contour.tanzu.vmware.com                             1.22.0+tap.3     Reconcile succeeded  
  conventions-controller              controller.conventions.apps.tanzu.vmware.com         0.7.1            Reconcile succeeded  
  developer-conventions               developer-conventions.tanzu.vmware.com               0.8.0            Reconcile succeeded  
  eventing                            eventing.tanzu.vmware.com                            2.0.1            Reconcile succeeded  
  fluxcd-source-controller            fluxcd.source.controller.tanzu.vmware.com            0.27.0+tap.1     Reconcile succeeded  
  grype                               grype.scanning.apps.tanzu.vmware.com                 1.3.0            Reconcile succeeded  
  image-policy-webhook                image-policy-webhook.signing.apps.tanzu.vmware.com   1.1.7            Reconcile succeeded  
  learningcenter                      learningcenter.tanzu.vmware.com                      0.2.3            Reconcile succeeded  
  learningcenter-workshops            workshops.learningcenter.tanzu.vmware.com            0.2.2            Reconcile succeeded  
  metadata-store                      metadata-store.apps.tanzu.vmware.com                 1.3.3            Reconcile succeeded  
  ootb-delivery-basic                 ootb-delivery-basic.tanzu.vmware.com                 0.10.2           Reconcile succeeded  
  ootb-supply-chain-testing-scanning  ootb-supply-chain-testing-scanning.tanzu.vmware.com  0.10.2           Reconcile succeeded  
  ootb-templates                      ootb-templates.tanzu.vmware.com                      0.10.2           Reconcile succeeded  
  policy-controller                   policy.apps.tanzu.vmware.com                         1.1.1            Reconcile succeeded  
  scanning                            scanning.apps.tanzu.vmware.com                       1.3.0            Reconcile succeeded  
  service-bindings                    service-bindings.labs.vmware.com                     0.8.0            Reconcile succeeded  
  services-toolkit                    services-toolkit.tanzu.vmware.com                    0.8.0            Reconcile succeeded  
  source-controller                   controller.source.apps.tanzu.vmware.com              0.5.0            Reconcile succeeded  
  spring-boot-conventions             spring-boot-conventions.tanzu.vmware.com             0.5.0            Reconcile succeeded  
  tap                                 tap.tanzu.vmware.com                                 1.3.0            Reconcile succeeded  
  tap-auth                            tap-auth.tanzu.vmware.com                            1.1.0            Reconcile succeeded  
  tap-gui                             tap-gui.tanzu.vmware.com                             1.3.0            Reconcile succeeded  
  tap-telemetry                       tap-telemetry.tanzu.vmware.com                       0.3.1            Reconcile succeeded  
  tekton-pipelines                    tekton.tanzu.vmware.com                              0.39.0+tap.2     Reconcile succeeded
```
