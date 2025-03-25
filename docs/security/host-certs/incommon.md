title: Requesting InCommon IGTF Host Certificates

Requesting InCommon IGTF Host Certificates
==========================================

Many institutions in the United States already subscribe to InCommon and offer IGTF certificate services.
If your institution is in the list of [InCommon subscribers](https://www.incommon.org/federation/incommon-federation-participants/),
continue with the instructions below.
If your institution is not in the list, Let's Encrypt certificates do not meet your needs, and you do not have access to
another IGTF CA subscription, please [contact us](../../common/help.md).


Before you can request a certificate, ensure your host has one of the following:

- The ability to run containers through tools like `docker` or `podman`
- An installation of the  `osg-pki-tools` RPM:
   - Select a host with [a supported operating system](../../release/supported_platforms.md).
   - Ensure root access on the host.
   - Configure the [required Yum repositories](../../common/yum.md).
   - Install the RPM via:
     ```console
     root@host # yum install osg-pki-tools
     ```

From a host that meets the above requirements, there are two options to get InCommon IGTF-accredited host certificates:

1. [Requesting certificates from a Registration Authority (RA)](#requesting-certificates-from-a-registration-authority):
   This requires a Certificate Signing Request (CSR), which can be generated with the `osg-cert-request` tool.
1. [Requesting certificates as an RA](#requesting-certificates-as-a-registration-authority):
   As an RA, you can request, approve, and retrieve certificates yourself through the InCommon REST API using the
   `osg-incommon-cert-request` tool .


## Requesting certificates from a registration authority

### Generating a CSR via the Container

1. Generate a Certificate Signing Request (CSR) and private key using the  `hub.opensciencegrid.org/opensciencegrid/pki-tools` container,
volume mounting a local directory (`~/path_to_dir`) into the `/output` directory of the container.

        :::console
        user@host $ docker run --rm  \
                     -v ~/path_to_dir:/output \
                     hub.opensciencegrid.org/opensciencegrid/pki-tools:24-release \
                     --hostname <HOSTNAME> \
                     --country <COUNTRY> \
                     --state <STATE> \
                     --locality <LOCALITY> \
                     --organization <ORGANIZATION>

    You may also add [DNS Subject Alternative Names](https://en.wikipedia.org/wiki/Subject_Alternative_Name) (SAN) to
    the request by specifying any number of `--altname <SAN>`.
    For example, the following generates a CSR for `test.opensciencegrid.org` with `foo.opensciencegrid.org` and
    `bar.opensciencegrid.org` as SANs:

        :::console
        user@host $ docker run --rm  \
                     -v ~/path_to_dir:/output \
                     hub.opensciencegrid.org/opensciencegrid/pki-tools:24-release \
                     --country US \
                     --state Wisconsin \
                     --locality Madison \
                     --organization 'University of Wisconsin-Madison' \
                     --hostname test.opensciencegrid.org \
                     --altname  foo.opensciencegrid.org \
                     --altname  bar.opensciencegrid.org

    If successful, the CSR will be named `~/path_to_dir/<HOSTNAME>.req` and the private key will be named 
    `~/path_to_dir/<HOSTNAME>-key.pem`.  Additional options and descriptions can be found 
    [here](https://github.com/opensciencegrid/osg-pki-tools#options).

### Generating a CSR via the RPM

1. Generate a Certificate Signing Request (CSR) and private key using the `osg-cert-request` tool:

        :::console
        user@host $ osg-cert-request --hostname <HOSTNAME> \
                     --country <COUNTRY> \
                     --state <STATE> \
                     --locality <LOCALITY> \
                     --organization <ORGANIZATION>

    You may also add [DNS Subject Alternative Names](https://en.wikipedia.org/wiki/Subject_Alternative_Name) (SAN) to
    the request by specifying any number of `--altname <SAN>`.
    For example, the following generates a CSR for `test.opensciencegrid.org` with `foo.opensciencegrid.org` and
    `bar.opensciencegrid.org` as SANs:

        :::console
        user@host $ osg-cert-request --hostname <HOSTNAME> \
                     --country US \
                     --state Wisconsin \
                     --locality Madison \
                     --organization 'University of Wisconsin-Madison' \
                     --hostname test.opensciencegrid.org \
                     --altname  foo.opensciencegrid.org \
                     --altname  bar.opensciencegrid.org

    If successful, the CSR will be named `<HOSTNAME>.req` and the private key will be named `<HOSTNAME>-key.pem`.
    Additional options and descriptions can be found [here](https://github.com/opensciencegrid/osg-pki-tools#options).


### Generating a Certificate using a CSR

Once you have obtained a CSR and private key using either the container or RPM:

1. Find your institution-specific InCommon contact and  submit the CSR that you generated above.
   Make sure to request a 1-year `IGTF Server Certificate` for `OTHER` server software.
1. After the certificate has been issued by your institution, download the host certificate only (not the full chain) to
   its intended host and copy over the key you generated above.
1. Verify that the issuer `CN` field is ` InCommon IGTF Server CA`:

        :::console
        $ openssl x509 -in <PATH TO CERTIFICATE> -noout -issuer
        issuer= /C=US/O=Internet2/OU=InCommon/CN=InCommon IGTF Server CA

    Where `<PATH TO CERTIFICATE>` is the file you downloaded in the previous step

1. Install the host certificate and key:

        :::console
        root@host # cp <PATH TO CERTIFICATE> /etc/grid-security/hostcert.pem
        root@host # chmod 444 /etc/grid-security/hostcert.pem
        root@host # cp <PATH TO KEY> /etc/grid-security/hostkey.pem
        root@host # chmod 400 /etc/grid-security/hostkey.pem

    Where `<PATH TO KEY>` is the ".key" file you created in the first step


## Requesting certificates as a registration authority

If you are a Registration Authority for your institution, skip ahead to [this section](#osg-incommon-cert-request).
If you are not already a Registration Authority (RA) for your institution, you must request to be made one:

1. Find your institution-specific InCommon contact (e.g. campus central IT)
1. Request a Department Registration Authority user with SSL auto-approve enabled and a client certificate:
    - If they do not grant your request, you will not be able to request, approve, and retrieve certificates yourself.
      Instead, you must [request certificates from your RA](#requesting-certificates-from-a-registration-authority).
    - If they grant your request, you will receive an email with instructions for requesting your client certificate;
      download the `.p12` file.
1. Find your institution-specific organization and department codes at the InCommon Cert Manager (https://cert-manager.com/customer/InCommon).
   These are numeric codes that should be specified through the command line using the -O/--orgcode ORG,DEPT option:

    * Organization code is shown as OrgID under Settings > Organizations > Edit
    * Department code is shown as OrgID under Settings > Organizations > Departments > Edit

Once you have RA privileges, you may request, approve, and retrieve host certificates using `osg-incommon-cert-request`:

<a name="osg-incommon-cert-request"></a>

-   In order to request a certificate, you will need your InCommon client certificate as two separate files,
    `incommon_user_key.pem` for the key, and `incommon_user_cert.pem` for the cert.
    If you don't already have them, perform the following steps:

    1. Download the `.p12` file with your client certificate and save this as `incommon_file.p12`.
       You should have received instructions for how to obtain this file in an email when you became an RA.

    1. Extract the certificate and key:

            :::console
            user@host $ openssl pkcs12 -in incommon_file.p12 \
                        -nocerts -out ~/path_to_dir/incommon_user_key.pem
            user@host $ openssl pkcs12 -in incommon_file.p12 \
                        -nokeys -out ~/path_to_dir/incommon_user_cert.pem

- Requesting a certificate with a single hostname `<HOSTNAME>`:

        :::console
        user@host $ osg-incommon-cert-request --username <INCOMMON_LOGIN> \
                    --cert ~/path_to_dir/incommon_user_cert.pem \
                    --pkey ~/path_to_dir/incommon_user_key.pem \
                    --hostname <HOSTNAME>
                    [--orgcode <ORG,DEPT>]

- Requesting a certificate with Subject Alternative Names (SANs):

        :::console
        user@host $ osg-incommon-cert-request --username <INCOMMON_LOGIN> \
                    --cert ~/path_to_dir/incommon_user_cert.pem \
                    --pkey ~/path_to_dir/incommon_user_key.pem \
                    --hostname <HOSTNAME> \
                    --altname <ALTNAME> \
                    --altname <ALTNAME2>
                    [--orgcode <ORG,DEPT>]

- Requesting certificates in bulk using a hostfile name:

        :::console
        user@host $ osg-incommon-cert-request --username <INCOMMON_LOGIN> \
                    --cert ~/path_to_dir/incommon_user_cert.pem \
                    --pkey ~/path_to_dir/incommon_user_key.pem \
                    --hostfile ~/path_to_file/hostfile.txt \
                    [--orgcode <ORG,DEPT>]

    Where the contents of `hostfile.txt` contain one hostname and any number of SANs per line:

        :::console
        hostname01.yourdomain
        hostname02.yourdomain hostnamealias.yourdomain hostname03.yourdomain
        hostname04.yourdomain hostname05.yourdomain


References
------------
-   [CILogon documentation for requesting InCommon certificates](http://www.cilogon.org/globus-with-incommon-ca)

-   [Useful OpenSSL commands (from NCSA)](http://security.ncsa.illinois.edu/research/grid-howtos/usefulopenssl.html) - e.g. how to convert the format of your certificate.

