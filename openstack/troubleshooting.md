# Troubleshooting

## NeutronAdminCredentialConfigurationInvalid

### Symptoms

```
Unexpected API Error. Please report this at http://bugs.launchpad.net/nova/ and attach the Nova API log if possible.
<class 'nova.exception.NeutronAdminCredentialConfigurationInvalid'> (HTTP 500) (Request-ID: req-9e1e2b43-b77f-4106-8e7b-ecd1c1cfa27e)
```

### Solution

1. Regenerate credentials
2. Restart Neutron and Nova containers:

    ```bash
    $ for i in $(docker ps |grep -i neutron |awk '{print $1}'); do echo $i; docker restart $i; done
    $ for i in $(docker ps |grep -i nova |awk '{print $1}'); do echo $i; docker restart $i; done
    ```
