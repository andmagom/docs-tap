# Configure the Tanzu Insight CLI plug-in

This topic explains how to configure the Tanzu Insight plug-in.

## <a id='set-tar-cert'></a>Set the target and certificate authority certificate

Note: These instructions are for the recommended configuration where ingress is enabled. For instructions on non-ingress setups, see [Configure target endpoint and certificate](../../scst-store/using-encrypted-connection.hbs.md#additional-resources) for more details.

{{> 'partials/insight-ingress-configuration' }}

## <a id='set-access-token'></a>Set the access token

{{> 'partials/insight-set-access-token' }}

## <a id='check-con'></a>Check the connection

Check that your configuration is correct and you are able to make a connection.

```console
tanzu insight health
```

For example:

```console
$ tanzu insight health
Success: Reached Metadata Store!
```
