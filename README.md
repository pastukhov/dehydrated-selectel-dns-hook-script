# Dehydrated Selectel DNS Hook Script
This is a Dehydrated hook script allows to pass dns-01 challenges with Selectel DNS hosting.
The advantage of the dns-01 challenge is that doesn't require a web server or even for the server 
to be Internet accessible. Only the DNS hosting needs to be Internet accessible and have an API. 
So you can even request a certificate from an internal server which you only use internally.

```
Processing test.example.com
 + Signing domains...
 + Creating new directory /etc/dehydrated/certs/test.example.com ...
 + Generating private key...
 + Generating signing request...
 + Requesting challenge for test.example.com...
Creating challenge record for test.example.com in zone example.com
Created record: '_acme-challenge.test.example.com. 60 IN TXT "m4rWqTgOdEV7vRlxYvBhGi0_0w4BewR8SvrirMmv_vo"'
Waiting for sync.............................
Completed
 + Responding to challenge for test.example.com...
Deleting challenge record for test.example.com from zone example.com
1 record sets deleted
 + Challenge is valid!
 + Requesting certificate...
 + Checking certificate...
 + Done!
 + Creating fullchain.pem...
 + Done!
```

The `hook.sh` script can me used in conjunction with [`dehydrated`](https://github.com/lukas2511/dehydrated) 
and Let's Encrypt's service (letsencrypt.org) to issue SSL certificates for domain names hosted in 
[Selectel](https://selectel.ru/). It is designed to be called by the `dehydrated` script to create and delete dns-01 
challenge records.

The script will automatically identify the correct domain zone for each domain name. It also supports 
certificates with alternative domain names in different domain zones.

## Dependencies

The script requires the following tools, all of which should be in your Linux distro.
- curl
- bash
- [jq](https://stedolan.github.io/jq/)
- sed
- xargs
- mailx (just for emailing errors)

This script will only work if `dehydrated` if using the following `config` settings.
```
CHALLENGETYPE="dns-01"
HOOK=hook.sh
HOOK_CHAIN="no"
```

Selectel DNS API requires an secret token (see corresponding [blog post](https://blog.selectel.ru/upravlenie-domenami-s-selectel-dns-api/) for details).

You need to create a scheduled task (e.g. cron) to ensure your certificates are renewed regularly (more details below).

## Scheduled Task for Renewals (cron)

Let's Encrypt certificate expire after 90 days, so you should create a cron task to check and renew the certificates 
if they will expire soon.

```
@daily /etc/dehydrated/dehydrated --cron  >/dev/null  #Check and renew SSL certificates from Let's Encrypt
```

## Restart Web Servers for Renewed Certificates 

Most services won't load any renewed certificate files until they are restarted or signaled to reload, sometimes 
because they have dropped privileges and can no longer read their configuration files. You can extend 
the `deploy_cert` function in the hook script restart services or copy/load certificates into the right places, 
whenever a new or renewed certificate is created. Below is a simple example to restart apache on CentOS/RHEL 6. 
Change the sudo commands to suit other distros. 

```
deploy_cert() {
    local DOMAIN="${1}" KEYFILE="${2}" CERTFILE="${3}" FULLCHAINFILE="${4}" CHAINFILE="${5}" TIMESTAMP="${6}"

    # Restart apache to read the new certificate files
    # Requires that user running dehydrated has sudoer rights to execute the right commands, e.g
    # dehydrated ALL = NOPASSWD: /sbin/service httpd configtest, /sbin/service httpd graceful

    # Only restart apache if the configuration is valid
    echo -n "Checking apache config: "
    sudo service httpd configtest
    if [[ $? -eq 0 ]]; then
      echo "Restarting apache to read the new certificate files for ${DOMAIN}"
      sudo service httpd graceful
    else
      (>&2 echo "Skipping restarting apache because apache config is invalid") 
    fi
}
```

## Limitations

`dehydrated` does not tell the hook script the challenge type, so this hook script has to assume every domain name 
is using a dns-01 challenge.
