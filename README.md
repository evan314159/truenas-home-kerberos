# truenas-home-kerberos
How I Setup Kerberos for a Home TrueNAS Environment

## Problem Statement
I wanted to use NFSv4 at home between TrueNAS and my Mac.

NFSv4 authentication is designed for enterprise environments.  The NFSv4 "simple / low infrastructure" system authentication mode (sec=sys) requires user ID numbers (uids) to match between TrueNAS and clients. It was designed for centrally-managed IT environments from decades ago. It is hard for an end user to implement themselves across multiple different platforms (TrueNAS, Mac, Linux each have different uid schemes).  It was also designed for a security model from decades ago -- any client can report they are any user ID number.

NFSv4 also supports Kerberos authentication (sec=krb5). Kerberos is a centralised authentication system that provides single sign-on for many protocols, including NFSv4 and SMB. It's how Windows devices seamlessly authenticate to SMB shares and other services in enterprise Windows environments. With Kerberos, NFSv4 does not require user IDs to match between TrueNAS and clients, making NFSv4 both easy to setup and secure.  Perfect!

TrueNAS doesn't come with a local Kerberos server or one in Apps.  Enterprises use Active Directory or FreeIPA for Kerberos, but they are complex for a small environment.  I wanted something simple for just a few users and computers.

## Approach
* Deploy Kerberos in a container running on TrueNAS
* Integrate TrueNAS and clients with Kerberos
* Setup NFSv4 to use Kerberos authentication
* Enable Kerberos for SMB, as an extra bonus

All TrueNAS configuration can be done with standard configurations via the TrueNAS GUI, except enabling Kerberos for SMB.

Kerberos for SMB should be enabled automatically by TrueNAS when a Kerberos server is configured, but in this case it is not.  Kerberos for SMB can be manually configured using non-standard SMB options via the CLI.

## Assumptions
- This is a non-production environment with no impact if things don't work now or in future. 
- TrueNAS is already deployed and working well.
- TrueNAS menus and commands are based on TrueNAS 25.04.1
- TrueNAS, Linux, Mac administration is well understood.
- TrueNAS Apps and containers are well understood.
- There is a local DNS service, or alternatively the environment uses TailScale MagicDNS.
- The threat model accepts that root access to TrueNAS also gives access to the Kerberos server and database.
- Changes to TrueNAS made outside of the GUI may not be supported by TrueNAS.

## Procedure
### Step 1: Basic Networking
Kerberos requires:

* Working hostname resolution, in both forwards (hostname -> IP address) and reverse (IP address -> hostname) directions for servers and sometimes clients as well.
* All the clients need to be able to reach the Kerberos server (the TrueNAS server).  Required ports are 88/udp, 88/tcp, 464/udp, 464/tcp.
* Hosts need a domain name. It does not need to be real. It cannot end in .local as this will conflict with multicast DNS.
* The Kerberos server, users, servers, and sometimes clients, will be members of a group called a Kerberos realm. The Kerberos realm is named after the domain name, in upper case.  Case matters!

This document will use:
* DNS domain: myhome.lan
* Kerberos realm: MYHOME.LAN
* TrueNAS server: mytruenas.myhome.lan
* User: myuser

I used TailScale and MagicDNS which meet all requirements and enable consistent access to the NFSv4 share even while remote.  If TailScale is used, the settings will be similar to:

* DNS domain: tailabc123.ts.net      # Tailnet name
* Kerberos realm: TAILABC123.TS.NET  # Tailnet name in capitals
* TrueNAS server: mytruenas.tailabc123.ts.net

A standalone internal DNS can be used instead, and maybe even hosts files in a small enough environment.  Setup of an internal DNS service or TailScale are outside the scope of this document.

If there is a firewall between the clients and the server, make sure Kerberos ports 88/udp, 88/tcp, 464/udp, 464/tcp are available, as well as 2049/tcp for NFSv4.

By the end of this step, verify that:
* Clients can ping the TrueNAS server
* Hostname lookup for the TrueNAS server successfully returns its IP address.
* IP address looking for the TrueNAS server successfully returns its fully-qualified hostname (mytruenas.myhome.lan).
* If both IPv4 and IPv6 are available then resolution for both IPv4 and IPv6 works.

### Step 2: Kerberos Container
1. Create container datasets:

TrueNAS recommends that production App datasets are created outside of the ix-apps dataset.  If there is not already an apps dataset, go to TrueNAS GUI -> Datasets -> select pool -> create dataset "apps" with dataset preset "Apps"

Create additional datasets apps/krb5kdc, apps/krb5kdc/data, apps/krb5kdc/build

2. Copy container build files:

Copy container build files from [Gabriel Abdalla Cavalcante's github](https://github.com/gcavalcante8808/docker-krb5-server) into apps/krb5kdc/build:
* [docker-entrypoint.sh](https://github.com/gcavalcante8808/docker-krb5-server/blob/master/docker-entrypoint.sh "docker-entrypoint.sh")
* [supervisord.conf](https://github.com/gcavalcante8808/docker-krb5-server/blob/master/supervisord.conf "supervisord.conf")*

Make the docker entrypoint script executable: chmod +x docker-entrypoint.sh

3. Create Dockerfile:
 
Create apps/krb5kdc/build/Dockerfile:
```
FROM alpine:latest
RUN apk add --no-cache krb5-server krb5 supervisor tini
ADD supervisord.conf /etc/supervisord.conf
ADD docker-entrypoint.sh /
VOLUME /var/lib/krb5kdc
EXPOSE 88 88/udp 464 464/udp
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["/docker-entrypoint.sh"]
```
This is slightly different from the Dockerfile in Gabriel Abdalla Cavalcante's github:
* Builds against current Alpine Linux image
* Exposes all the ports used by Kerberos clients
* Does not expose the Kerberos administration port

4. Deploy the container:

Go to TrueNAS GUI -> Apps -> Discovery Apps -> ... -> Install from YAML.  Update the YAML below to suit the environment.  Additional port 794/tcp for remote administration is not exposed for security reasons.

```
services:
  krb5kdc:
    build: /mnt/path/to/apps/krb5kdc/build ## update the apps path
    environment:
      KRB5_KDC: mytruenas.myhome.lan
      KRB5_REALM: MYHOME.LAN
    ports:
      - '88:88'
      - 88:88/udp
      - '464:464'
      - 464:464/udp
    restart: unless-stopped
    volumes:
      - /mnt/path/to/apps/krb5kdc/data:/var/lib/krb5kdc ## update the apps path
```
5. Get the admin password from the container log and write it down for future use: docker logs ix-krb5kdc-krb5kdc-1

### Step 3: Setup Kerberos Principals
1. ssh to the TrueNAS server.

2. Enter the Kerberos container: docker exec -it ix-krb5kdc-krb5kdc-1 /bin/sh

3. Start the Kerberos administration tool: kadmin.local

4. Add principals for the TrueNAS server and export to a keytab file:
```
Create the host principal:
kadmin.local: addprinc -randkey host/mytruenas.myhome.lan@MYHOME.LAN

Create the NFS principal:
kadmin.local: addprinc -randkey nfs/mytruenas.myhome.lan@MYHOME.LAN

Create the SMB/CIFS principal:
kadmin.local: addprinc -randkey cifs/mytruenas.myhome.lan@MYHOME.LAN

Export keys to the keytab, stored in /var/lib/krb5kdc which is available on the host at apps/krb5kdc/data

kadmin.local: ktadd -k /var/lib/krb5kdc/mytruenas.keytab host/mytruenas.myhome.lan@MYHOME.LAN
kadmin.local: ktadd -k /var/lib/krb5kdc/mytruenas.keytab nfs/mytruenas.myhome.lan@MYHOME.LAN
kadmin.local: ktadd -k /var/lib/krb5kdc/mytruenas.keytab cifs/mytruenas.myhome.lan@MYHOME.LAN
```
Note: every time a key is exported the key version number increments. The installed keytab must have the latest version of the keys.

5. For any Linux clients, repeat the above procedure only for the host principal:
```
Create the host principal:
kadmin.local: addprinc -randkey host/mylinux.myhome.lan@MYHOME.LAN
kadmin.local: ktadd -k /var/lib/krb5kdc/mylinux.keytab host/mylinux.myhome.lan@MYHOME.LAN
```
6. Macs do not need a keytab to act as a NFS or SMB client.

7. Create user principals for the users:
```
Create the user principal:
kadmin.local: addprinc@MYHOME.LAN
Enter the user's password when prompted.

Repeat for as many users as required.
Do not export the user keys to a keytab.
```
8. Securely copy the keytab files from the TrueNAS server apps/krb5kdc/data dataset to the TrueNAS web administration client.  Remove the copies from the server.

### Step 4: Configure TrueNAS to use Kerberos
1. In the TrueNAS GUI go to Credentials -> Directory Services -> Advanced Settings -> Show.

2. Add the Kerberos Realm:

Next to kerberos Realms click Add:
```
Realm: MYHOME.LAN
KDC: mytruenas.myhome.lan
Admin Servers: mytruenas.myhome.lan
Password Servers: mytruenas.myhome.lan
Click Save
```

3. Add Kerberos default settings:

Next to Kerberos Settings click Add:
```
Libdefaults Auxiliary Parameters:
default_realm = MYHOME.LAN
dns_lookup_realm = false # use the krb5.conf to find the
dns_lookup_kdc = false   # realm and Kerberos server
Click Save
```
4. Upload the Kerberos keytab.

Next to Kerberos Keytab click Add:
```
Name: mytruenas
Choose file -> mytruenas.keytab
Click Save
```

### Step 5: Enable NFSv4 with Kerberos
1. In the TrueNAS GUI go to Shares -> UNIX (NFS) Shares -> ... -> Config Service.

2. Set as follows:
```
Enabled Protocols: NFSv4
NFSv4 DNS Domain: myhome.lan
Require Kerberos for NFSv4: checked
Allow non-root mount: checked (especialy for Mac clients)
Manage Groups Server-side: checked
```
3. Setup an NFS share as desired.  Under Advanced Options, Kerberos security levels can be configured.  The Kerberos security levels are:
```
KRB5: Kerberos authentication only
KRB5I: Kerberos authentication with Kerberos integrity (packets cannot be changed in transit but are not encrypted in transit)
KRB5P: Kerberos authentication with Kerberos encryption (packets cannot be changed in transit and are encrypted in transit)
```
The Kerberos security level set on the client mount option (sec=krb5|krb5i|krb5p) needs to match an enabled Kerberos security level on TrueNAS.  The default (none selected) is all Kerberos security levels are allowed. sec=sys is disabled by global option "Require Kerberos fo NFSv4".

### Step 6: Configure a Mac client
1. Create /etc/krb5.conf permissions root:wheel 0644 to contain:
```
[libdefaults]
 default_realm = MYHOME.LAN
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 MYHOME.LAN = {
    kdc = mytruenas.myhome.lan
    admin_server = mytruenas.myhome.lan
 }
```
2. Update /etc/nfs.conf to use NFSv4.0 and Kerberos authentication by default:
```
nfs.client.default_nfs4domain = myhome.lan
nfs.client.mount.options = vers=4.0,sec=krb5
# consider further options such as soft,intr,timeo,retrans based on the environment
```
3. Get the Kerberos Ticket Granting Ticket:
```
kinit myuser@MYHOME.LAN  # get the Ticket Granting Ticket
klist                    # shows the ticket

Other options:
kinit --keychain will save the Kerberos credentials in the Mac Keychain, making it easier to get tickets going forward

kinit (no parameters) will kinit for current-mac-user@DEFAULT_REALM which may be the same as myuser@MYHOME.LAN, and if available will use the Kerberos credentials stored in Keychain
```
4. Mount the NFS share:
```
In Finder click Go -> Connect to Server
Server path: nfs://mytruenas.myhome.lan/mnt/full/path/to/share
```
5. The share will be mounted and should be browsable as per usual.  The share will be available under sudo root.

6. After every reboot the user will need to kinit to get a a new Kerberos Ticket Granting Ticket.

### Step 7: Configure Linux Client
1. Linux requires certain services and libraries to be installed and running for NFS and Kerberos.  I used Fedora 42 which had everything i needed already installed and running.

2. Create /etc/krb5.conf, permissions root:root 0644:
```
[libdefaults]
 default_realm = MYHOME.LAN
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 MYHOME.LAN = {
    kdc = mytruenas.myhome.lan
    admin_server = mytruenas.myhome.lan
 }
```
3. Linux clients require a keytab because the kernel does the initial NFS mount using the equivalent of a machine account.  Install the Linux client keytab in /etc/krb5.keytab, permissions root:root 600.

4. Reboot after installing /etc/krb5.keytab to enable Kerberos kernel modules.

5. From the user account, get the Kerberos Ticket Granting Ticket:
```
kinit myuser@MYHOME.LAN  # get the Ticket Granting Ticket
klist                    # shows the ticket

Other options:
kinit (no parameters) will kinit for current-linux-user@DEFAULT_REALM which may be the same as myuser@MYHOME.LAN
```
6. Mount the share:
```
sudo mkdir /run/media/share
sudo mount -t nfs -o vers=4.2,sec=krb5 mytruenas.myhome.lan:/full/path/to/share /run/media/share
```
7. The share should be mounted and browseable by the user as per normal.  Note that the share will only be available for users with a Kerberos ticket. The NFS ticket is visible through klist.

8. To mount the share permanently, add an /etc/fstab entry similar to:
```
mytruenas.myhome.lan:/full/path/to/share /run/media/share -o vers=4.2,sec=krb5 0 0
```
9. After every reboot the user will need to kinit to get a new Kerberos Ticket Granting Ticket.

### Step 8: Bonus: Kerberos for SMB
SMB is designed to use Kerberos in Windows domain environments, but Kerberos is only available with SMB on TrueNAS in AD-integrated configurations.

Kerberos can be enabled for SMB through the command line using the existing Kerberos configuration.

1. ssh to TrueNAS server.

2. Open TrueNAS cli and configure smb_options
```
Warning: the supported mechanisms for making configuration changes
are the TrueNAS WebUI, CLI, and API exclusively. ALL OTHERS ARE
NOT SUPPORTED AND WILL RESULT IN UNDEFINED BEHAVIOR AND MAY
RESULT IN SYSTEM FAILURE.

Welcome to TrueNAS
Last login: Tue Jun 24 15:18:47 2025 from 2403:580f:5827:0:16:5b4f:aa4a:b51f
root@fileserver[~]# cli

[mytruenas]> service smb
[mytruenas] service smb> config
... returns your current SMB config

[mytruenas] service smb> update smb_options="security = user\nrealm = MYHOME.LAN\nkerberos method = system keytab\nkerberos encryption types = strong"

[mytruenas] service smb> config
...
|     smb_options | security = user                         |
|                 | realm = TAIL3F4EB5.TS.NET               |
|                 | kerberos method = system keytab         |
|                 | kerberos encryption types = strong      |
```
If there are already have non-standard smb_options these will also need to be included in the updated smb_options. See smb.conf(5), but also be aware that TrueNAS has restricted access to most settings for a reason.

3. Mount the share. Use klist to the Kerberos ticket that was obtained for the share.

4. Access to the share will remain available via NTLMv2 the user is disabled for SMB or NTLMv2 is disabled via smb_options.

## Backup and Recovery
I am snap-shotting the Kerberos apps/krb5kdc/data directory have tested roll-back snapshots if required, and then backing up the snapshots as per usual. It is possible that the snapshots are not crash-consistent but my Kerberos database is minimal and changes very infrequently. If there is ever an irrecoverable issue with Kerberos, Kerberos can be reset by deleting the contents of the data dataset and restarting Kerberos.

## Troubleshooting
* Basic network troubleshooting: is the TrueNAS server reachable, do forwards and reverse DNS work
* Is the Kerberos container running, and are there errors in the log: docker logs ix-krb5kdc-krb5kdc-1
* Is there a Kerberos Ticket Granting Ticket: klist
* Are there service tickets for NFS or SMB: klist
* Linux clients require the keytab in /etc/krb5.keytab, permissions root:root 0600.

## Final Thoughts
I hope this will inspire others to use secure NFSv4 authentication, where they otherwise would not have the infrastructure to support. In my case, TailScale made this much easier by providing a managed DNS service and static IPs in an otherwise fully-dynamic environment, and also faciliates remote access from anywhere.

I hope this will also inspire TrueNAS to include a local Kerberos server in the base install to handle local authentication for NFSv4 and SMB -- like Microsoft is doing on Windows so that NTLMv2 can be deprecated. Samba and MIT Kerberos support for Windows-compatible local Kerberos (IAKERB) is coming soon and I hope will be quickly integrated into TrueNAS and clients.

To learn more about Kerberos, TheSecMaster has a very detailed explanation here: https://thesecmaster.com/blog/what-is-kerberos-comprehensive-step-by-step-guide-to-understanding-how-kerberos-authentication-works

Thank you to Gabriel Abdalla Cavalcante for his excellent Kerberos server build scripts, available from his github https://github.com/gcavalcante8808/docker-krb5-server.
