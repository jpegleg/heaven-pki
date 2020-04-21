# heaven-pki
A private PKI tool with identity tracking component generation using ECC, not RSA.

A guided setup script (you can do the setup on your own based on your CA requirements):

sudo bash making-heaven_initial_CA_setup


After initial setup, then you can actually run the 
angel-daemon and identity programs as non-priveledged users.
You could make a user for each daemon and easily customize
work directories to match your filesystem desires.

Decrypting a file delivered by the angel demon can be done multiple ways.
If you are doing it on the command line or in an automated fashion, here
is an example approach:

  HISTCONTROL=ignoreboth  

  heavenfile=/var/heaven-pki/landing-zone/$(hostname).identity.tbin.tgz.asc

  gpg --yes --no-tty --passphrase $GPXENV -d $heavenfile > /var/heaven-pki/identity.tbin.tgz 

  idfile=/var/heaven-pki/landing-zone/$(hostname).identity-file.asc

  gpg --yes --no-tty --passphrase $GPXENV -d $idfile > /var/heaven-pki/identity-file.txt 



# Way to use it #1 (identity-build):

Make a CA server that has SSH configured with keys to allow access to
each server that receives an identity file.

cp identity-build /usr/local/bin/
chmod +x /usr/local/bin/identity-build

The CA server runs the Heaven PKI "identity-build" with the input.list file
containing all of the servers, we'll call them "subjects" in Heaven,
line delimited:

khost1.com
khost2.com
somethingsomething
10.1.1.45


The angel-daemon runs on the CA server, shipping out the identity files
to each subject over SSH, as configured in /etc/heaven-subjects.cfg
which is also a line delimited file that might look like this:

pkideploy@khost1.com
pkideploy@khost2.com
root@somethingsomething
transport-admin@10.1.1.45


Then the CA server has crontab entries to schedule identity renewals
by executing the identity-build before the certificate expires. 
The default length of the issued certificates is 1 year.


# Way to use it #2

cp identity-api-backend /usr/local/bin/
chmod +x /usr/local/bin/identity-api-backend

Instead of crontabs and making the identities in a batch, use API
calls. You will have to build/create the front-end on your own
or hire me to make one for you. ( carefuldata at protonmail dot com )

The identity-api-backend is called by a front end, or on the command line.
Note that it does not have a locking mechanism built in, the front end
must control that. Because of that, if the  	identity-api-backend is 
called twice at the same time, they can corrupt eachothers processing!

The  	identity-api-backend creates an identity from the first argument
passed to it. It takes the FQDN in base64 encoded format. 

Manual example:

makeidentity=$(echo -n khost1.com | base64)
identity-api-backend $makeidentity


And then the angel-daemon picks it up and sends it if configured in
the /etc/heaven-subjects.cfg over to the host.

# Way to use it #3

Use the identity-api-backend2, which is a wrapper for the identity-build, but will take in 
standard web server log URI contexts. This way your front end can pass the HTTPD/NGINX data to
the backend as the argument. 

This method adds in deriving a shared secret from a public secp384r1 key in the 7th. Example URI:

yourcaserver:/pki/api/id/gen/$base64FQDN/$base64secp384r1public

So it would actually look something like this, using some example public key and FQDN:

https://yourcaserver-url:/pki/api/id/gen/ZXhhbXBsZW5vZGUxLm9yZw==/LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUhZd0VBWUhLb1pJemowQ0FRWUZLNEVFQUNJRFlnQUVZTlZiN3gzODd6V0RmQzRsSlZtSHVCRFJUY2pNaEFIVQp0Q0lNZFZtVVNmc0ZFQkJ4VklZcmhhRi9EWWo1VlZNaSt5SFZSOFZWeWRIOFVMNWs1aW8xeG1TUHJrdCsydTZUClc5QitDTmhMUlEreDB5MmtOWkZpZ2poOWt4TUhqS0swCi0tLS0tRU5EIFBVQkxJQyBLRVktLS0tLQo=

To control which devices are able to make api requests, use firealling and TLS client authentication on your front end if you take this route.

The subject (client) will need the ca servers public key if it is going to derive the secret to decrypt its contents from
the angel daemon. You can have the front end publish the ca server EC public keys so that the subject can curl it down etc.

# More about the shared secret (EC derived or not, the GPXENV)

There is a shared secret between the subjects and the Heaven PKI CA server that is 
used via environment variable. This choice was made to be compatible with ephemeral
infrastructure and to not need the secret to be shared on the disk of each subject,
but stored in memory.

Typically the subjects will be actually using the ECC cert and key pair and
sharing their public key to derive shared secrets. This can be done for GPXENV
as well:


gensecp () {
  openssl ecparam -name secp384r1 -genkey -noout -out secp384r1.pem
  openssl ec -in secp384r1.pem -pubout -out secp384r1.pub  
}

derive () {
  openssl pkeyutl -derive -inkey secp384r1.pem -peerkey peerpub.pem -out secret.bin  
}


gensecp will give you a fresh secp384r1 curve private and public key. The subjects
could have a key pair like this that is shared across all of them, before setting
up Heaven, and then the Heaven CA server/s have another key pair like this.
The subjects and the CA server exchange the public parameters, renaming the remote
public key to peerpub.pem.

Then run the derive function on each host to use ECC to derive a shared secret,
and set GPXENV to that!

# And don't forget about history logs, don't log GPXENV when you don't want to!

Because exporting environment variables in command line can be logged to shell history logging,
keep in mind measures to avoid storing the GPXENV value in the logs:


HISTCONTROL=ignoreboth
`

And then keep two spaces before your command :)
