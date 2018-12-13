# A Walk through of Key Rotation of a HashiCorp VAULT cluster using AWS KMS to AutoUnseal

## PGP (Keybase) is used to encrypt the recovery keys

``` bash
ubuntu@ip-192-168-100-194:~$ export VAULT_ADDR=http://127.0.0.1:8200

ubuntu@ip-192-168-100-194:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       awskms
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  n/a

ubuntu@ip-192-168-100-194:~$ cat /etc/vault.d/vault.hcl
```

``` hcl
storage "file" {
  path = "/opt/vault"
}
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}
seal "awskms" {
  region     = "eu-west-1"
  kms_key_id = "df57391c-dd5a-47fd-867f-aaea4cf0911f"
}
ui=true
```

Initialise Vault and supply the list of PGP keys

``` bash

ubuntu@ip-192-168-100-194:~$ vault operator init -recovery-shares=1 -recovery-threshold=1 -recovery-pgp-keys="keybase:grahamhashicorp"
Recovery Key 1: wcBMA7YWwEEOUNNlAQgAWonI80EyrhCfEnBdBpgbub+nRdUXga1SMpJExcgNd4kM28Fxhnui29MRtiaJVkVoraD+cQ2d4kRWQaTtMynu16ANHD0dUl5r6tBWxPLzfbXFBagxKIQnHIUYQZ1KriOzAJ9DqG15BCtZJLtKpwSlvXIGN+y9w6QNBEJeHYfe/FIaDDJ/l3AmfYO2ak17qBmI4GTeFdAWxg+5DuepuUokQD+jGcw5pV2sj+vFzEzx1fJDkNYJyYLRVfDLsvT5jHx2SyGL7f6B3tK+yFZi/Qt8bZszPWxvs0AqiY4cxlRqfqHyDDQvtA+ooso5lZoy6AwSWAKMHE7MPu/NjkXQ+0q8ONLgAeS/zYrVFfrPoAc8pBYjCrmF4VSL4NHg4eEnXeAv4t6PxiDg4OaxSbJMzvQwRDiCCR1UdObJFkSrWiEd422PKShIFKQDg5X0jcBNh9xgfv4F3TDyvm4pXTgbdB0HRNbB36SDfN+Z4CvkDq+90nQ+eAogDuUiQxJTWeIgUew34QD9AA==

Initial Root Token: s.JdUbF6aSl93Mrc325IRgf8jc

Success! Vault is initialized

Recovery key initialized with 1 key shares and a key threshold of 1. Please
securely distribute the key shares printed above.
```

Restart the cluster to verify that the AutoUnseal mechanism is working

``` bash
ubuntu@ip-192-168-100-194:~$ sudo systemctl restart vault

ubuntu@ip-192-168-100-194:~$ vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    1
Threshold                1
Version                  1.0.0
Cluster Name             vault-cluster-df5b2f4f
Cluster ID               4a863a26-e3dd-ad19-cc11-e0c773e99e2f
HA Enabled               false
```

_Rekey Process - Recovery Keys_

In order to rekey Vault's recovery keys it's necessary it's necessary to decode the PGP wrapped key first.
I've stored the pgp wrapped key in `recovery-key.txt`

``` bash
Grahams-MacBook-Pro:aws_autounseal grazzer$ cat recovery-key.txt | base64 --decode | keybase pgp decrypt

6be6f42f89c9d50aab7d17a4fe19c667bc19ffb1bde01ca696b1c8240a1d9352
```

The above key needs to be entered when prompted for the recovery keys (in production obviously you will use more than one key...please)
Start the rekey process for the recovery key(s)

``` bash
ubuntu@ip-192-168-100-194:~$ vault operator rekey -target=recovery -init -pgp-keys=keybase:grahamhashicorp -key-shares=1 -key-threshold=1
WARNING! You are using PGP keys for encrypted the resulting unseal keys, but
you did not enable the option to backup the keys to Vault's core. If you lose
the encrypted keys after they are returned, you will not be able to recover
them. Consider canceling this operation and re-running with -backup to allow
recovery of the encrypted unseal keys in case of emergency. You can delete the
stored keys later using the -delete flag.

Key                      Value
---                      -----
Nonce                    b13f1ad7-a59d-ddc2-ae8e-f52518569a9b
Started                  true
Rekey Progress           0/1
New Shares               1
New Threshold            1
Verification Required    false
PGP Fingerprints         [4bb84487f2c3be2b3e61704cd8f67af593e82e66]
Backup                   false
```

Each keyholder will need to enter their respective decoded (PGP) key

``` bash

ubuntu@ip-192-168-100-194:~$ vault operator rekey -target=recovery -nonce=b13f1ad7-a59d-ddc2-ae8e-f52518569a9b
Rekey operation nonce: b13f1ad7-a59d-ddc2-ae8e-f52518569a9b
Unseal Key (will be hidden):

Key 1 fingerprint: 4bb84487f2c3be2b3e61704cd8f67af593e82e66; value: wcBMA7YWwEEOUNNlAQgAWrD5hAx9/JENYp3Wxu/Wdxn6wbVtb1wCrwxV1m0mw5SRIiz1Ls8I4mxGOLIF810T149ij22PgMUqFTshE0K+aBuyYvnKUlXq14uR1L85LXRP6pEwbcuGhUt4/1TqxftAP0/ZMghbtHvUvci/wrvgPZhe4bI6jXlmKC1krSiBGLly4vSj6YilXajfW03L3h2N4FZgfvW3wdE7hyFPcrI9l7dFZ/KWn5Mzvbh7SKOIIG38sEi/UkdseXDH4W5Y5wvZVasyn0JkjdPJ7b1NKqxjq3/e/Ivszp6xZ7T/vwarsT/EADggFJZ7VZ/nOqKm2A4f6P71WX8A07HYBa+LM41oNNLgAeQGuN6iVhXf3DivSK1j7xit4eN+4AHgWeE7l+BP4t/MT8/gnuZ+1wPBxbKjui5uRvIR575Nzc0w/kGgwVkAL/wb08kNem2DSOHZ7r7vhcRlsrw6oSVAeUjzb/4GlIIm1pwh88J/4JPkkyyiPf/rA06X3MXAhk6RGOJLSVj84f5wAA==

Operation nonce: b13f1ad7-a59d-ddc2-ae8e-f52518569a9b

Vault rekeyed with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When Vault is re-sealed, restarted,
or stopped, you must supply at least 1 of these keys to unseal it before it
can start servicing requests.

```

Now they need to safely store their new PGP wrapped keys identified above.

_Rotate the Encryption Key_
Along with rekeying Vault it is also possible to rotate the encryption key.
This is a much simpler process you'll be pleased to note.

``` bash
ubuntu@ip-192-168-100-194:~$ export VAULT_TOKEN=s.JdUbF6aSl93Mrc325IRgf8jc

ubuntu@ip-192-168-100-194:~$ vault operator rotate
Success! Rotated key

Key Term        2
Install Time    13 Dec 18 14:02 UTC
ubuntu@ip-192-168-100-194:~$
```