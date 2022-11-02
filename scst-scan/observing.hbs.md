# Observability and troubleshooting

This section outlines observability and troubleshooting methods and issues for using the Supply Chain Security Tools - Scan components.

## <a id="observability"></a> Observability

The scans  run inside a Kubernetes Job where the Job creates a pod. Both the Job and pod are cleaned up after completion.

Before applying a new scan, you can set a watch on the Jobs, Pods, SourceScans, Imagescans to observe their progression:
```console
watch kubectl get scantemplates,scanpolicies,sourcescans,imagescans,pods,jobs
```

## <a id="troubleshooting"></a> Troubleshooting

### <a id="debugging-commands"></a> Debugging commands

Run these commands to get more logs and details about the errors around scanning. The Jobs and pods persist for a predefined amount of seconds before getting deleted. (`deleteScanJobsSecondsAfterFinished` is the tap pkg variable that defines this)

####  <a id="debugging-scan-pods"></a> Debugging Scan pods

Run the following to get error logs from a pod when scan pods are in a failing state:

```console
kubectl logs scan-pod-name -n DEV-NAMESPACE
```

See [here](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_logs/) for more details about debugging Kubernetes pods.

The following is an example of a successful scan run output:
```yaml
scan:
  cveCount:
    critical: 20
    high: 120
    medium: 114
    low: 9
    unknown: 0
  scanner:
    name: Grype
    vendor: Anchore
    version: v0.37.0
  reports:
  - /workspace/scan.xml
eval:
  violations:
  - CVE node-fetch GHSA-w7rc-rwvf-8q5r Low
store:
  locations:
  - https://metadata-store-app.metadata-store.svc.cluster.local:8443/api/sources?repo=hound&sha=5805c6502976c10f5529e7f7aeb0af0c370c0354&org=houndci
```

A scan run that has an error means that one of the init containers: `scan-plugin`, `metadata-store-plugin`, `compliance-plugin`, `summary`, or any other additional containers had a failure.

To inspect for a specific init container in a pod:

```console
kubectl logs scan-pod-name -n DEV-NAMESPACE -c init-container-name
```

See [Debug Init Containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-init-containers/) in the Kubernetes documentation for debug init container tips.

####  <a id="debug-source-image-scan"></a> Debugging SourceScan and ImageScan

To retrieve status conditions of an SourceScan and ImageScan, run:

```console
kubectl describe sourcescan <sourcescan> -n DEV-NAMESPACE
```

```console
kubectl describe imagescan <imagescan> -n DEV-NAMESPACE
```

Under `Status.Conditions`, for a condition look at the "Reason", "Type", "Message" values that use the keyword "Error" to investigate issues.

#### <a id="debug-scanning-in-supplychain"></a> Debugging Scanning within a SupplyChain

See [here](../cli-plugins/apps/debug-workload.md) for Tanzu workload commands for tailing build and runtime logs and getting workload status and details.


#### <a id="view-scan-controller-manager-logs"></a> Viewing the Scan-Controller manager logs

To retrieve scan-controller manager logs:
```console
kubectl -n scan-link-system logs -f deployment/scan-link-controller-manager -c manager
```

### <a id="restarting-deployment"></a> Restarting Deployment

If you encounter an issue with the scan-link controller not starting, run the following to restart the deployment to see if it's reproducible or flaking upon starting:

```console
kubectl rollout restart deployment scan-link-controller-manager -n scan-link-system
```

### <a id="troubleshooting-issues"></a> Troubleshooting issues

#### <a id="miss-src-ps"></a> Missing target SSH secret

Scanning source code from a private source repository requires an SSH secret present in the namespace and referenced as `grype.targetSourceSshSecret` in `tap-values.yaml`. See [Installing the Tanzu Application Platform Package and Profiles](../install.md).

If a private source scan is triggered and the secret cannot be found, the scan pod includes a `FailedMount` warning in Events with the message `MountVolume.SetUp failed for volume "ssh-secret" : secret "secret-ssh-auth" not found`, where `secret-ssh-auth` is the value specified in `grype.targetSourceSshSecret`.

#### <a id="miss-img-ps"></a> Missing target image pull secret

Scanning an image from a private registry requires an image pull secret to exist in the Scan CRs namespace and be referenced as `grype.targetImagePullSecret` in `tap-values.yaml`. See [Installing the Tanzu Application Platform Package and Profiles](../install.md).

If a private image scan is triggered and the secret is not configured, the scan job fails with the error as follows:

```console
Job.batch "scan-${app}-${id}" is invalid: [spec.template.spec.volumes[2].secret.secretName: Required value, spec.template.spec.containers[0].volumeMounts[2].name: Not found: "registry-cred"]
```

#### <a id="deactivate-scst-store"></a> Deactivate Supply Chain Security Tools (SCST) - Store

SCST - Store is a prerequisite for installing SCST - Scan.
If you install without the SCST - Store, you must edit the
configurations to deactivate the Store:

  ```yaml
  ---
  metadataStore:
    url: ""
  ```

  Install the package with the edited configurations by running:

  ```console
  tanzu package install scan-controller \
    --package-name scanning.apps.tanzu.vmware.com \
    --version VERSION \
    --namespace tap-install \
    --values-file tap-values.yaml
  ```

#### <a id="incompatible-syft-schema-version"></a> Resolving Incompatible Syft Schema Version

  You might encounter the following error:

  ```console
  The provided SBOM has a Syft Schema Version which doesn't match the version that is supported by Grype...
  ```

  This means that the Syft Schema Version from the provided SBOM doesn't match the version supported by the installed `grype-scanner`. There are two different methods to resolve this incompatibility issue:

  - (Preferred method) Install a version of [Tanzu Build Service](../tanzu-build-service/tbs-about.md) that provides an SBOM with a compatible Syft Schema Version.
  - Deactivate the `failOnSchemaErrors` in `grype-values.yaml`. See [Install Supply Chain Security Tools - Scan](install-scst-scan.md). Although this change bypasses the check on Syft Schema Version, it does not resolve the incompatibility issue and produces a partial scanning result.

    ```yaml
    syft:
      failOnSchemaErrors: false
    ```

#### <a id="incompatible-scan-policy"></a> Resolving incompatible scan policy
  If your scan policy appears to not be enforced, it might be because the Rego file defined in the scan policy is incompatible with the scanner that is being used. For example, the Grype Scanner outputs in the CycloneDX XML format while the Snyk Scanner outputs SPDX JSON.

  See [Install Snyk Scanner](install-snyk-integration.md#a-idverifya-verify-integration-with-snyk) for an example of a ScanPolicy formatted for SPDX JSON.

#### <a id="ca-not-found-in-secret"></a> Could not find CA in secret

  If you encounter the following issue, it might be due to not exporting  `app-tls-cert` to the correct namespace:

  ```console
  {"level":"error","ts":"2022-06-08T15:20:48.43237873Z","logger":"setup","msg":"Could not find CA in Secret","err":"unable to set up connection to Supply Chain Security Tools - Store"}
  ```

  Configure `ns_for_export_app_cert` in your `tap-values.yaml`.

  ```yaml
  metadata_store:
    ns_for_export_app_cert: "DEV-NAMESPACE"
  ```

  If there are multiple developer namespaces, use `ns_for_export_app_cert: "*"`.

#### <a id="reporting-wrong-blob-url"></a> Blob Source Scan is reporting wrong source URL

A Source Scan for a blob artifact can cause reporting in the `status.artifact` and `status.compliantArtifact` the wrong URL for the resource, passing the remote SSH URL instead of the cluster local fluxcd one. One symptom of this issue is the `image-builder` failing with a `ssh:// is an unsupported protocol` error message.

You can confirm you're having this problem by running a `kubectl describe` in the affected resource and comparing the `spec.blob.url` value against the `status.artifact.blob.url` and see if they are different URLs. For example:

```console
kubectl describe sourcescan SOURCE-SCAN-NAME -n DEV-NAMESPACE
```

And compare the output:

```console
...
spec:
  blob:
    ...
    url: http://source-controller.flux-system.svc.cluster.local./gitrepository/sample/repo/8d4cea98b0fa9e0112d58414099d0229f190f7f1.tar.gz
    ...
status:
  artifact:
    blob:
      ...
      url: ssh://git@github.com:sample/repo.git
  compliantArtifact:
    blob:
      ...
      url: ssh://git@github.com:sample/repo.git
```

**Workaround:** This problem happens in SCST - Scan `v1.2.0` when you use a Grype Scanner ScanTemplates earlier than `v1.2.0`, because this is a deprecated path. To fix this problem, upgrade your Grype Scanner deployment to `v1.2.0` or later. See [Upgrading Supply Chain Security Tools - Scan](upgrading.md#upgrade-to-1-2-0) for step-by-step instructions.

#### <a id="supply-chain-stops"></a> Resolving failing scans that block a Supply Chain 

If the Supply Chain is not progressing due to CVEs found in either the SourceScan or ImageScan, see the CVE triage workflow in [Triaging and Remediating CVEs](triaging-and-remediating-cves.hbs.md).

#### <a id="gui-miss-policy"></a> Policy not defined in the Tanzu Application Platform GUI

If you encounter `No policy has been defined`, it might be because the Tanzu Application Platform GUI is unable to view the Scan Policy resource.

Confirm that the Scan Policy associated with a SourceScan or ImageScan exists. For example, the `scanPolicy` in the scan matches the name of the Scan Policy.

```
kubectl describe sourcescan NAME -n DEV-NAMESPACE
kubectl describe imagescan NAME -n DEV-NAMESPACE
kubectl get scanpolicy NAME -n DEV-NAMESPACE
```

Add the `app.kubernetes.io/part-of` label to the Scan Policy. See [Enable Tanzu Application Platform GUI to view ScanPolicy Resource](policies.hbs.md#a-idgui-view-scan-policyaenable-tanzu-application-platform-gui-to-view-scanpolicy-resource) for more details.
