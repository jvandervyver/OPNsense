Update Supermicro IPMI TLS certificate

ACME client should already be installed and the IPMI certificate bein renewed by ACME client

As root on OPNsense

```
mkdir '/usr/local/supermicro'
chmod 755 '/usr/local/supermicro'
chown root:wheel '/usr/local/supermicro'

curl --output '/usr/local/supermicro/ipmi-updater.py' 'https://gist.githubusercontent.com/mattisz/d112ebfe1869c56ce111ecbd2cbbd04d/raw/569b20ddc8bcc2c04a875de2e9e918570a0cf93a/ipmi-updater.py'
chmod 755 '/usr/local/supermicro/ipmi-updater.py'
chown root:wheel '/usr/local/supermicro/ipmi-updater.py'
```

```
touch '/usr/local/supermicro/update-ipmi-cert.sh'
chmod 755 '/usr/local/supermicro/update-ipmi-cert.sh'
chown root:wheel '/usr/local/supermicro/update-ipmi-cert.sh'
```

Contents of `update-ipmi-cert.sh`
```
# No easy way to find my, simply search '/var/etc/acme-client/cert-home/' sub-directories until the certificate home is found
CERT_NUMBER="176dddeb53e764.17682947"
IPMI_URL='123.456.789.012'
MOTHERBOARD_MODEL='X12'
FQDN='ipmi.mydomain.com'
CERT_PATH="/var/etc/acme-client/cert-home/${CERT_NUMBER}/${FQDN}"
KEY_PATH="${CERT_PATH}/${FQDN}.key"
CER_PATH="${CERT_PATH}/${FQDN}.cer"
USERNAME='IPMI username'
PASSWORD='IPMI password'

/usr/local/supermicro/ipmi-updater.py --ipmi-url "https://${IPMI_URL}" --model "${MOTHERBOARD_MODEL}" --key-file "${KEY_PATH}" --cert-file "${CER_PATH}" --username "${USERNAME}" --password "${PASSWORD}" --force-update
```

Add automation action for ACME renewal:

```
touch /usr/local/opnsense/service/conf/actions.d/actions_ipmi.conf
chmod 644 /usr/local/opnsense/service/conf/actions.d/actions_ipmi.conf
chown root:wheel /usr/local/opnsense/service/conf/actions.d/actions_ipmi.conf
```

Contents of `actions_ipmi.conf`
```
[cert]
command: /usr/sbin/daemon -f /usr/local/supermicro/update-ipmi-cert.sh
type:script
message:Updating IPMI TLS certificate
description:Update IPMI TLS certificate
```

Now create the automation in ACME Client **Automations**
![Create automation](https://raw.githubusercontent.com/jvandervyver/OPNsense/refs/heads/main/ipmi/automation_pic.jpg)

