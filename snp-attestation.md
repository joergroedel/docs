# SNP Guest Attestation

To use the hardware provided launch attestation mechanisms of an AMD SEV-SNP
guest, the [snpguest](https://github.com/virtee/snpguest) tool is required.

The following steps will download the necessary keys and certificates must be
downloaded first.

## Download the ASK and ARK Certificates

These certificates are unique per AMD EPYC processor generation and are the
root of trust from the certificate chain perspective. The certificates can be
downloaded from the [AMD web-site](https://www.amd.com/en/developer/sev.html) or
with the `snpguest` utility:

```
# snpguest fetch ca Milan certs/
```

`Milan` is the EPYC generation for with the certificates should be fetched,
valid values are `Naples`, `Rome`, `Milan`, and `Genoa`. Only `Milan` and
`Genoa` support SEV-SNP.

The `certs/` parameter specifies a directory to store the certificates in. It
will be created if it does not exist yet.

```
# ls -l certs/
total 8
-rw-r--r-- 1 root root 2277 Dec 14 14:34 ark.pem
-rw-r--r-- 1 root root 2325 Dec 14 14:34 ask.pem
```

These certificates are unique per EPYC processor generation. This step does not
need to be done on the SNP guest VM, but the EPYC generation needs to match the
host hardware which the SNP VM is running on.

## Generate an Attestation Report

The next step is to generate an attestation report (AR). The AR is bound to the
running VM and needs to be regenerated and re-validated after reboot of the VM.

Further, to prevent replay attacks the AR contains a nonce to prove its
freshness. For the purpose of this document we will let the `snpguest` utility
generate a nonce. In real-world setups the nonce should be provided by the
relying party.

To generate an AR, use this command:

```
# snpguest report --random
# ls -l
total 8
-rw-r--r-- 1 root root 1184 Dec 14 14:47 attestation_report.bin
drwxr-xr-x 1 root root   28 Dec 14 14:34 certs
-rw-r--r-- 1 root root  131 Dec 14 14:47 random-request-file.txt

```

This will create two new files, `attestation_report.bin` which contains the AR
in binary format and `random-request-file.txt` containing a list of generated
nonce.

This step must be done in the SNP guest VM that is about to be validated. The AR
can then be transmitted to the relying party for verification.

## Download the VECK

With the attestation report the Versioned Chip Endorsement Key (VECK) can be
fetched from AMDs KDS. The following command will do the job:

```
# snpguest fetch vcek Milan certs/
# ls -l certs/
total 12
-rw-r--r-- 1 root root 2277 Dec 14 14:34 ark.pem
-rw-r--r-- 1 root root 2325 Dec 14 14:34 ask.pem
-rw-r--r-- 1 root root 1361 Dec 14 14:49 vcek.der
```

After that command the VECK is stored in the `certs/` directory.

## Verifying the Attestation Report

For verifying the AR the singing key needs to be verified first to establish
trust into the VECK. The following command will do the verification:

```
# snpguest verify certs certs/
The AMD ARK was self-signed!
The AMD ASK was signed by the AMD ARK!
The VCEK was signed by the AMD ASK!
```

If the output looks like above, the VECK successfully verified using the
certificate chain starting at the AMD ARK.

Now that the VECK is verified and trusted, the signature of the AR can be
verified:

```
# snpguest verify signature certs/
VCEK signed the Attestation Report!
```

This output indicates that the signature of the attestation report was made
using the trusted VECK. This proves a number of things:

* The VM is running with AMD SEV-SNP protection
* The VM is running on real AMD hardware and not in an emulated environment

This is a strong cryptographic proof that the VM is running with Confidential
Computing protection. The verification can be done at the relying party and does
not need to be done within the SNP guest VM.
