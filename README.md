# Encrypted Secrets
*Keep it secret, keep it safe.*

Problem: Keeping passwords, api keys, or any secret safe while maintaining use of chosen tool set.

*Largely copied from: https://gist.github.com/shadowhand/873637*

An entire git repository will be used to store secrets.  Most contents of the repository will be encrypted, exceptions will be defined in a `.gitattributes` file.

For a complete end to end solution to deploy secrets in a CI environment, see: https://gist.github.com/ckelner/54c17aa5c1b44650bfeb

## Table of Contents
- [Initial Setup](#initial-setup): How to configure a machine to initialize the repository.
- [Repository Setup](#repository-setup): How to setup the repository for encryption.
- [Using the Repository](#using-the-repository): How to perform a fresh clone of the repository.
- [Comitting Secrets](#comitting-secrets): How to commit back to the repository.
- [Continuous Integration](#continuous-integration): How to use this repo with CI

### Initial Setup

Before creating the repository, the machine which will initialize the repository must be setup first.  Perform the following commands in user's home directory `~/`:
```shell
mkdir .gitencrypt
cd !$
touch clean_filter_openssl smudge_filter_openssl diff_filter_openssl
chmod -R 700 ~/.gitencrypt
```

`clean_filter_openssl` will encrypt the contents of the repository.

`smudge_filter_openssl` will decrypt the contents of the repository.

`diff_filter_openssl` will be used to compare files when using `git diff`.

Contents of `clean_filter_openssl`:
```bash
#!/bin/bash

SALT_FIXED=<your-salt> # 24 or less hex characters
PASS_FIXED=<your-passphrase>

openssl enc -base64 -aes-256-ecb -S $SALT_FIXED -k $PASS_FIXED
```

Replace `<your-salt>` with 24 or less hex characters and `<your-passphrase>` with a passphrase to be used as a secret for the symmetric key encryption and decryption. This script uses AES-256 ECB mode as the encryption algorithm, as it turns out a deterministic encryption works best with git.  The reason non-deterministic encryption (what GPG does) does not work very well with git is because the same file is transformed to a different ciphertext each time it is encrypted.  Therefore when performing a `git status` it will always shows the pulled files at modified, even though a `git diff` shows no difference. Checking in such modified files only replaces the old ciphertext with a new one which decrypts to the same file. If work is being done in two different local repositories synced to the same remote, the push/pull process will never end even if nothing is changed in the working directories. Using AES ECB mode with a fixed salt, although not semantically secure, resolves this problem while providing reasonable confidentiality.

See https://www.openssl.org/docs/apps/enc.html for more details about the encoding command being used here.

Contents of `smudge_filter_openssl`:
```bash
#!/bin/bash

# No salt is needed for decryption.
PASS_FIXED=<your-passphrase>

# If decryption fails, use `cat` instead. 
# Error messages are redirected to /dev/null.
openssl enc -d -base64 -aes-256-ecb -k $PASS_FIXED 2> /dev/null || cat
```

Replace `<your-passphrase>` with the passphrase used in `clean_filter_openssl`.

Contents of `diff_filter_openssl`:
```bash
#!/bin/bash

# No salt is needed for decryption.
PASS_FIXED=<your-passphrase>

# Error messages are redirect to /dev/null.
openssl enc -d -base64 -aes-256-ecb -k $PASS_FIXED -in "$1" 2> /dev/null || cat "$1"
```

Replace `<your-passphrase>` with the passphrase used in `clean_filter_openssl`.

Files in the `.gitencrypt` directory should only be stashed locally and never shared with any person or system that should not have access to these secrets.

### Repository Setup

The repository needs to be configured to use these `.gitencrypt` scripts as defined in "Machine Setup".  Create the git repository in GitHub then clone to desired directory.

Navigate to the new repo then open `.git/config` and add the following to the end of the file where `filter` and `diff` attributes are assigned to drivers named `openssl`:
```
[filter "openssl"]
    smudge = ~/.gitencrypt/smudge_filter_openssl
    clean = ~/.gitencrypt/clean_filter_openssl
[diff "openssl"]
    textconv = ~/.gitencrypt/diff_filter_openssl
```

In the project directory create and open `.gitattributes` and add the following content:
```
* filter=openssl diff=openssl
README.md -filter -diff
.gitattributes -filter -diff
[merge]
    renormalize=true
```

The first line indicates to encrypt all files, but the second and third lines exclude the `.gitattributes` and `README.md` files from encryption.  The merge `renormalize` entry prevents changes caused by check-in conversion from causing spurious merge conflicts when a converted file is merged with an unconverted file.

An example of the filters in action:
```
$ date > top-secret.txt

$ cat top-secret.txt 
Fri Mar 13 07:44:13 EDT 2015

$ git add -A

$ git commit -m "testing encryption"
[master 5ba77e4] testing encryption
 1 file changed, 1 insertion(+)
 create mode 100644 top-secret.txt

$ git ls-tree head
100644 blob c5dc31c6ffef2ff94e09e3ad8a13b0431fc0f22a  .gitattributes
100644 blob 512702ac63c5c055885b2c9623b18eaddceeaca1  README.md
100644 blob 6176564bcfcaec9bc6b541f59ba3cb3e4fe04ca5  top-secret.txt

$ git cat-file -p 6176564bcfcaec9bc6b541f59ba3cb3e4fe04ca5
U2FsdGVkX1+a3AjzgWsEp4ddqvRZmGc11JSpl2uhlzPEwZVrz/Y546pAa78olsXN

$ cat top-secret.txt 
Fri Mar 13 07:44:13 EDT 2015
```

As seen in the example, the staged file, and ultimately what gets pushed to the repo, is encrypted, while the local copy is not.  When a new user checks out the repo for the first time the smudge filter runs to decrypt the files.

### Using the Repository

To use the repository on any other machine, or a fresh `git clone`, special steps must be taken for encryption to work.

1) The intial `git clone` must be performed with the `-n` flag.  `-n` doesn't perform a checkout of HEAD after the git clone is complete.

```
$ git clone -n https://github.com/ckelner/encrypted-secrets.git
Cloning into 'encrypted-secrets'...
Username for 'https://github.com': 
Password for 'https://<user>@github.com': 
remote: Counting objects: 80, done.
remote: Compressing objects: 100% (55/55), done.
remote: Total 80 (delta 20), reused 67 (delta 10), pack-reused 0
Unpacking objects: 100% (80/80), done.
Checking connectivity... done.
```

2) Configure `.git/config`:
```
cd encrypted-secrets/
vi .git/config
```

The following snippit should be appended to `.git/config` as seen in the section "Machine Setup":
```
[filter "openssl"]
    smudge = ~/.gitencrypt/smudge_filter_openssl
    clean = ~/.gitencrypt/clean_filter_openssl
[diff "openssl"]
    textconv = ~/.gitencrypt/diff_filter_openssl
```

3) Then `.gitattributes` will need to be added manually and a [copy can be found in this repo](https://github.com/ckelner/encrypted-secrets/blob/master/.gitattributes).
```
$ vi .gitattributes
```
`.gitattributes` contents:
```
* filter=openssl diff=openssl
.gitattributes -filter -diff
README.md -filter -diff
gitconfig -filter -diff
clean_filter_openssl -filter -diff
diff_filter_openssl -filter -diff
smudge_filter_openssl -filter -diff
[merge]
    renormalize=true
```

4) Now perform `git reset --hard HEAD` - this will bring in the latest from GitHub and the repository contents will be fetched then decrypted.

#### Comitting Secrets

One cavet to this method is that the GitHub apps cannot be used to commit changes.  At the time of writing this (2015/03/16) they do not support the use of filters in `.git/config` or settings in `.gitattributes`.

**Caution** should be exercised when using this repository with regards to GitHub apps, because the GitHub app will commit all files as unencrypted (including those you had not altered locally due to the `smudge` filters).

A typical commit using the command line looks like:
```
git add -A #or some variation thereof
git commit -m "your commit message"
git push
```

## Continuous Integration
See: [CONTINUOUS_INTEGRATION.md](https://github.com/ckelner/encrypted-secrets/blob/master/CONTINUOUS_INTEGRATION.md)
