---
layout: "docs"
page_title: "SSH Secret Backend"
sidebar_current: "docs-secrets-ssh"
description: |-
  The SSH secret backend for Vault generates dynamic SSH keys or One-Time-Passwords.
---

# SSH Secret Backend

Name: `ssh`

Vault SSH backend dynamically generates SSH credentials for remote hosts. This
increases security by removing the need to share private keys with all users
needing access to infrastructure. It also solves the problem of management and
distribution of keys belonging to remote hosts.

This backend supports two types of credential creation: Dynamic Key and
One-Time Password (OTP), which address these problems in different ways.

Read and carefully understand both of them before choosing the one which best
suits your needs. The Vault team strongly recommends the OTP type whenever
possible, and the drawbacks to the dynamic key type should be carefully
considered before choosing it.

This page will show a quick start for this backend. For detailed documentation
on every path, use `vault path-help` after mounting the backend.

### Mounting SSH

The `ssh` backend is not mounted by default and needs to be explicitly mounted.
This is a common step for both OTP and Dynamic Key types.

```text
$ vault mount ssh
Successfully mounted 'ssh' at 'ssh'!
```

----------------------------------------------------
## I. One-Time-Password (OTP) Type

This backend type allows a Vault server to issue an OTP every time a client
wants to SSH into a remote host, using a helper command on the remote host to
perform verification.

An authenticated client requests credentials from the Vault server and, if
authorized, is issued an OTP. When the client establishes an SSH connection to
the desired remote host, the OTP used during SSH authentication is received by
the Vault helper, which then validates the OTP with the Vault server. The Vault
server then deletes this OTP, ensuring that it is only used once.

Since the Vault server is contacted during SSH connection establishment, every
login attempt and the correlating Vault lease information is logged to the
audit backend.

See [Vault-SSH-Helper](https://github.com/hashicorp/vault-ssh-helper) for
details on the helper.

### Drawbacks

The main concern with the OTP backend type is the remote host's connection to
Vault; if compromised, an attacker could spoof the Vault server returning a
successful request. This risk can be mitigated by using TLS for the connection
to Vault and checking certificate validity; future enhancements to this backend
may allow for extra security on top of what TLS provides.

### Creating a Role

Create a role with the `key_type` parameter set to `otp`. All of the machines
represented by the role's CIDR list should have helper properly installed and
configured.

```text
$ vault write ssh/roles/otp_key_role \
    key_type=otp \
    default_user=username \
    cidr_list=x.x.x.x/y,m.m.m.m/n
Success! Data written to: ssh/roles/otp_key_role
```

### Create a Credential

Create an OTP credential for an IP of the remote host that belongs to
`otp_key_role`.

```text
$ vault write ssh/creds/otp_key_role ip=x.x.x.x
Key            	Value
lease_id       	ssh/creds/otp_key_role/73bbf513-9606-4bec-816c-5a2f009765a5
lease_duration 	600
lease_renewable	false
port           	22
username       	username
ip             	x.x.x.x
key            	2f7e25a2-24c9-4b7b-0d35-27d5e5203a5c
key_type       	otp
```

### Establish an SSH session

```text
$ ssh username@localhost
Password: <Enter OTP>
username@ip:~$
```

### Automate it!

A single CLI command can be used to create a new OTP and invoke SSH with the
correct parameters to connect to the host.

```text
$ vault ssh -role otp_key_role username@x.x.x.x
OTP for the session is `b4d47e1b-4879-5f4e-ce5c-7988d7986f37`
[Note: Install `sshpass` to automate typing in OTP]
Password: <Enter OTP>
```

The OTP will be entered automatically using `sshpass` if it is installed.

```text
$ vault ssh -role otp_key_role -strict-host-key-checking=no username@x.x.x.x
username@<IP of remote host>:~$
```

Note: `sshpass` cannot handle host key checking. Host key checking can be
disabled by setting `-strict-host-key-checking=no`.

----------------------------------------------------
## II. Dynamic Key Type

When using this type, the administrator registers a secret key with appropriate
`sudo` privileges on the remote machines; for every authorized credential
request, Vault creates a new SSH key pair and appends the newly-generated
public key to the `authorized_keys` file for the configured username on the
remote host. Vault uses a configurable install script to achieve this.

The backend does not prompt for `sudo` passwords; the `NOPASSWD` option for
sudoers should be enabled at all remote hosts for the Vault administrative
user.

The private key returned to the user will be leased and can be renewed if
desired. Once the key is given to the user, Vault will not know when it gets
used or how many time it gets used. Therefore, Vault **WILL NOT** and cannot
audit the SSH session establishments.

When the credential lease expires, Vault removes the secret key from the remote
machine.

### Drawbacks

The dynamic key type has several serious drawbacks:

1. _Audit logs are unreliable_: Vault can only log when users request
   credentials, not when they use the given keys. If user A and user B both
   request access to a machine, and are given a lease valid for five minutes,
   it is impossible to know whether two accesses to that user account on the
   remote machine were A, A; A, B; B, A; or B, B.
2. _Generating dynamic keys consumes entropy_: Unless equipped with a hardware
   entropy generating device, a machine can quickly run out of entropy when
   generating SSH keys. This will cause further requests for various Vault
   operations to stall until more entropy is available, which could take a
   significant amount of time, after which the next request for a new SSH key
   will use the generated entropy and cause stalling again.

Because of these drawbacks, the Vault team recommends use of the OTP type
whenever possible. Care should be taken with respect to the above issues with
any deployments using the dynamic key type.

### sudo

In order to adjust the `authorized_keys` file for the desired user, Vault
connects via SSH to the remote machine as a separate user, and uses `sudo` to
gain the privileges required. An example `sudoers` file is shown below.

File: `/etc/sudoers`

```hcl
# This is a sample sudoers statement; you should modify it
# as appropriate to satisfy your security needs.
vaultadmin   ALL=(ALL)NOPASSWD: ALL
```

### Configuration

Next, infrastructure configuration must be registered with Vault via roles.
First, however, the shared secret key must be specified.

#### Registering the shared secret key

Register a key with a name; this key must have administrative capabilities on
the remote hosts.

```text
$ vault write ssh/keys/dev_key \
    key=@dev_shared_key.pem
```

#### Create a Role

Next, create a role. All of the machines contained within this CIDR block list
should be accessible using the registered shared secret key.

```text
$ vault write ssh/roles/dynamic_key_role \
    key_type=dynamic \
    key=dev_key \
    admin_user=username \
    default_user=username \
    cidr_list=x.x.x.x/y
Success! Data written to: ssh/roles/dynamic_key_role
```

`cidr_list` is a comma separated list of CIDR blocks for which a role can
generate credentials. If this is empty, the role can only generate credentials
if it belongs to the set of zero-address roles.

Zero-address roles, configured via `/ssh/config/zeroaddress` endpoint, takes
comma separated list of role names that can generate credentials for any IP
address.

Use the `install_script` option to provide an install script if the remote
hosts do not resemble a typical Linux machine. The default script is compiled
into the Vault binary, but it is straight forward to specify an alternate.  The
script takes three arguments which are explained in the comments.

To see the default, see
[linux_install_script.go](https://github.com/hashicorp/vault/blob/master/builtin/logical/ssh/linux_install_script.go)

### Create a credential

Create a dynamic key for an IP of the remote host that is covered by
`dynamic_key_role`'s CIDR list.

```text
$ vault write ssh/creds/dynamic_key_role ip=x.x.x.x
Key            	Value
lease_id       	ssh/creds/dynamic_key_role/8c4d2042-23bc-d6a8-42c2-6ff01cb83cf8
lease_duration 	600
lease_renewable	true
ip             	x.x.x.x
key            	-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA5V/Y95qfGaUXRPkKNK9jgDHXPD2n5Ein+QTNnLSGrHtJUH7+
pgs/5Hc4//124P9qHNmjIYQVyvcLreFgSrQCq4K8193hmypBYtsvCgvpc+jEwaGA
zK0QV7uc1z8KL7FuRAxpHJwB6+nubOzzqM03xsViHRhaWhYVHw2Vl4oputSHE7R9
ugaTRg67wge4Nyi5RRL0RQcmW15/Vop8B6HpBSmZQy3enjg+32KbOWCMMTAPuF9/
DgxSgZQaFMjGN4RjDreZI8Vv5zIiFJzZ3KVOWy8piI0PblLnDpU4Q0QSQ9A+Vr7b
JS22Lbet1Zbapl/n947/r1wGObLCc5Lilu//1QIDAQABAoIBAHWLfdO9sETjHp6h
BULkkpgScpuTeSN6vGHXvUrOFKn1cCfJPNR4tWBuXI6LJM2+9nEccwXs+4IMwjZ0
ZfVCdI/SKtZxBXmP2PxBGMUMP7G/mn0kN64sDlD3ezOvQZgZVEmZFpCrvixYsG+v
qlpZ+HhrlJEWds7tvBsyyfNjwWjVIpm08zBmteFj4zu7OEcmGXEHDoxDXxyVP2BG
eLU/fM5JA2UEjfCQ1MIZ3rBtPePdz4LRpb+ajklqrUj1OHoiDrXa8EAf0/wDP9re
c1iH4bn7ZjYK0+IhZ+Pmw6gUftzZNWSC2kOLnZLdN/K7hgh0l0r0K/1eeXt43upB
WALNuiECgYEA8PM2Ob3XXKALF86PUewne4fCz9iixr/cIpvrEGrh9lyQRO8X5Jxb
ug38jEql4a574C6TSXfzxURza4P6lnfa0LvymmW0bhxZ5nev9kcAVnLKvpOUArTR
32k9bKXd6zp8Q9ZyVNwHRxcVs4YgwfJlcx8geC4o6YRiIjvcBQ9RVHkCgYEA87OK
lZDFBeEY/HVOxAQNXS5fgTd4U4DbwEJLv7SPk02v9oDkGHkpgMs4PcsIpCzsTpJ0
oXMfLSxZ1lmZiuUvAupKj/7RjJ0XyjSMfm1Zs81epWj+boVfM4amZNHVLIWgddmM
XzXEZKByvi1gs7qFcjQz2DEbZltWO6dX14O4Fz0CgYEAlWSWyHJWZ02r0xT1c7vS
NxtTxH7zXftzR9oYgtNiStfVc4gy7kGr9c3aOjnGZAlFMRhvpevDrxnj3lO0OTsS
5rzBjM1mc6cMboLjDPW01eTSpBroeE0Ym0arGQQ2djSK+5yowsixknhTsj2FbfsW
v6wa+6jTIQY9ujAXGOQIbzECgYAYuXlw7SwgCZNYYappFqQodQD5giAyEJu66L74
px/96N7WWoNJvFkqmPOOyV+KEIi0/ATbMGvUUHCY36RFRDU9zXldHJQz+Ogl+qja
VsvIAyj8DSfrHJrpBlsxVVyUVMZPzo+ARVs0flbF1qK9+Ul6qbMs1uaZvuCD0tmF
ovZ1XQKBgQDB0s7SDmAMgVjG8UBZgUru9vsDrxERT2BloptnnAjSiarLF5M+qeZO
7L4NLyVP39Z83eerEonzDAHHbvhPyi6n2YmnYhGjeP+lPZIVqGF9cpZD3q48YHZc
3ePn2/oLZrXKWOMyMwp2Uj+0SArCW+xMnoNp50sYNVR/JK3BPIdkag==
-----END RSA PRIVATE KEY-----
key_type       	dynamic
port           	22
username       	username
```

### Establish an SSH session

Save the key to a file (e.g. `dyn_key.pem`) and then use it to establish an SSH
session.

```text
$ ssh -i dyn_key.pem username@<IP of remote host>
username@<IP of remote host>:~$
```

### Automate it!

Creation of new key, saving to a file, and using it to establish an SSH session
can all be done with a single Vault CLI command.

```text
$ vault ssh -role dynamic_key_role username@<IP of remote host>
username@<IP of remote host>:~$
```

----------------------------------------------------
## III. CA Key Type

When using this type, an SSH key is generated and then used to sign other SSH
keys.  The public half of the key is distributed to remote hosts while the
private part stays within Vault. This allows SSH public keys to be signed by
Vault and then verified using only the public key.

### Configure a CA certificate

The first thing to do is to get Vault to generate the key pair that will be
used to sign any SSH keys:

```text
$ vault write -f ssh/config/ca
Key       	Value
---       	-----
public_key	ssh-rsa AAAAHHNzaC1yc2EtY2VydC12MDFAb3BlbnNzaC5jb20AAAAgxSlUi1Fd38w93emsotVQBjLYorkQTmCyRo0XPxJw/poAAAADAQABAAABAQCgbXubSftRY1JFEfFpkoHkf/4WkGNQr8g+X1H8kcU/UJUoFZl5IXaZrDzRUTUUQsC3bZA6EPerqSlgpy9gSYn/dtGcCCoPyOUQpaz3vRbF180ddzJnjaJvIAg1PHecFFLC+WjCPFeGkZPc5Yr1NyGhL5GiMUbv5fIYfSM5REkydcEn5+fryfZq8ZCSNBa0KfHflWvy9Nn3i3ns1ZphkMPp+DRkGw0Iy4VetfvUWd3bbVRP8PMZOz0o9Bo/90qzST3qBJ6DZip9LehBXfoNk3dvD/Rkst4IdjBLVv/gHnwX9V0yG8NMUCHh695S0anNtbjCFW1JedYXH7h5ayGOPfivg0P4QLigJ6cAAAABAAAABHJvb3QAAAAAAAAAAFhfuTMAAAAAWF/xkQAAAAAAAAAAAAAAAAAAARcAAAAHc3NoLXJzYQAAAAMBAAEAAAEBALiUMk0TnJh++UOYEU6LcsRAxTcZbR31XbbvtXBGLdK9P92ufZuSxvASVjEoHiJuI+a+rnw7q4GGwoBZQ4wooN/Az5Iy7ez04sz629UINQgUfHbp8RHVk3tCBrJ1F0aQKNEDz3LKNNuAF6kJZrXZ2d0pdCDorm0cNfaYZxOmyKAQtVH454xR2gP0VYUwOWcxTPF8lnoNecL6drEKxg0eyGl2dK+MndsE2TwE9b1S2LDatzfmVzVKQWL5JJWgNwGNiy65E0C858TLzQ7imrVqPomp3SppWLItMUNHZgy9uujyS3BeMqzLT6e1e+ndWMD92Ei2/t95JaSR9IMmClQS0BkAAAEPAAAAB3NzaC1yc2EAAAEAM4vtt9WhBtB98XfJsVo5TXI+XU6aAXm/yZH8wRpCl3ghhBDk5ZFdZredLna2v8jYELTNJGt8LuFZVy7XoXgsPC58kwhWcYx2BbtN3GpBDijlG7Odozwf03RrJ48LgheI9UfF+8mituwrerQDYppPgW5tws+THllhcWD099LU+iDvuC69aVEDy+CZJZKBvaYVDQYtu5bVlqdlGo5KE1ASro2h/jLQG2atl4iwpQ7NKi5VF5YuNFNX9NsWFIqnm5ErwXLdroBJb/XOSSWNE8Vlsi+UhNRJ33o3/QwQ3nMAjyxh1btnv2HW0r4Z3D4a63r+HizFP+RrGdRzNf7xj9UiRw==
```

### Creating a Role

The next step is to configure a role. A role is a logical name that maps to a
policy used to generate those credentials. For example, let's create an
"example" role:

```text
$ vault write ssh/roles/example ttl=4h allow_user_certificates=true key_type=ca
Success! Data written to: ssh/roles/example
```

### Create a Credential

By writing to the `roles/example` path we are defining the `example` role. To
sign an SSH public key, we simply write to the `sign` end point with that role
name: Vault is now configured to create and manage SSH certificates!

```text
$ cat dummy.pub | vault write ssh/sign/example public_key=-
Key             Value
---             -----
lease_id        ssh/sign/example/3c3740ee-6066-55c0-4a5d-82a544a474a3
lease_duration  768h0m0s
lease_renewable false
serial_number   8343f840b8a027a7
signed_key      ssh-rsa-cert-v01@openssh.com AAAAHHNzaC1yc2EtY2VydC12MDFAb3BlbnNzaC5jb20AAAAgxSlUi1Fd38w93emsotVQBjLYorkQTmCyRo0XPxJw/poAAAADAQABAAABAQCgbXubSftRY1JFEfFpkoHkf/4WkGNQr8g+X1H8kcU/UJUoFZl5IXaZrDzRUTUUQsC3bZA6EPerqSlgpy9gSYn/dtGcCCoPyOUQpaz3vRbF180ddzJnjaJvIAg1PHecFFLC+WjCPFeGkZPc5Yr1NyGhL5GiMUbv5fIYfSM5REkydcEn5+fryfZq8ZCSNBa0KfHflWvy9Nn3i3ns1ZphkMPp+DRkGw0Iy4VetfvUWd3bbVRP8PMZOz0o9Bo/90qzST3qBJ6DZip9LehBXfoNk3dvD/Rkst4IdjBLVv/gHnwX9V0yG8NMUCHh695S0anNtbjCFW1JedYXH7h5ayGOPfivg0P4QLigJ6cAAAABAAAABHJvb3QAAAAAAAAAAFhfuTMAAAAAWF/xkQAAAAAAAAAAAAAAAAAAARcAAAAHc3NoLXJzYQAAAAMBAAEAAAEBALiUMk0TnJh++UOYEU6LcsRAxTcZbR31XbbvtXBGLdK9P92ufZuSxvASVjEoHiJuI+a+rnw7q4GGwoBZQ4wooN/Az5Iy7ez04sz629UINQgUfHbp8RHVk3tCBrJ1F0aQKNEDz3LKNNuAF6kJZrXZ2d0pdCDorm0cNfaYZxOmyKAQtVH454xR2gP0VYUwOWcxTPF8lnoNecL6drEKxg0eyGl2dK+MndsE2TwE9b1S2LDatzfmVzVKQWL5JJWgNwGNiy65E0C858TLzQ7imrVqPomp3SppWLItMUNHZgy9uujyS3BeMqzLT6e1e+ndWMD92Ei2/t95JaSR9IMmClQS0BkAAAEPAAAAB3NzaC1yc2EAAAEAM4vtt9WhBtB98XfJsVo5TXI+XU6aAXm/yZH8wRpCl3ghhBDk5ZFdZredLna2v8jYELTNJGt8LuFZVy7XoXgsPC58kwhWcYx2BbtN3GpBDijlG7Odozwf03RrJ48LgheI9UfF+8mituwrerQDYppPgW5tws+THllhcWD099LU+iDvuC69aVEDy+CZJZKBvaYVDQYtu5bVlqdlGo5KE1ASro2h/jLQG2atl4iwpQ7NKi5VF5YuNFNX9NsWFIqnm5ErwXLdroBJb/XOSSWNE8Vlsi+UhNRJ33o3/QwQ3nMAjyxh1btnv2HW0r4Z3D4a63r+HizFP+RrGdRzNf7xj9UiRw==
```

### Establish an SSH session

Save the key to a file (e.g. `dummy-cert.pem`) and then use it to establish an
SSH session.

```text
$ ssh -i dummy.pem username@<IP of remote host>
username@<IP of remote host>:~$
```

----------------------------------------------------
## API

The SSH secret backend has a full HTTP API. Please see the
[SSH secret backend API](/api/secret/ssh/index.html) for more
details.
