---
title: "Passwords Over GnuPG"
date: 2021-10-27T21:04:22+05:30
description: "Manage your passwords using gpg and git"
---

## Passwords are dangerous

Identity on a multi-tenant service, at some point in time, requires you to prove
your identity in some form. Passwords are supposed to be restricted knowledge
which should be possessed by only a single person. That's the ideal case and is
also true in some cases. With the expanding online lives of humans and
dependence on multi-tenant platforms, it's becoming harder and harder for humans
to keep track of that "restricted" knowledge. It's a very risky practice to use
the same passwords at multiple locations for obvious reasons. In case one
platform is compromised, by extension, the other is too. It's also not good to
use similar or derivable passwords depending on the platform.

> Passwords were meant for machines not humans

Creating and remembering complicated and distinct passwords are not something
which humans are good at. Humans are good at deriving and observing patterns,
something that is not good for password creation.

## Solution to passwords

Machines are dumb too, they can only function according to the instructions they
were giving which makes them not a good candidate for generating passwords. But
over the years, machines have got a lot better in generating pseudo-randomness for
cryptographic purposes. This article will refer to password generation and
random number generation interchangeably because they are essentially all bits
and can be interpreted in any fashion.

So, let the machines generate passwords for your accounts and distinct and random
passwords with no correlations. Easier said than done.

Platforms are always on the look for methods to generate streams of bits with as
much [entropy](https://en.wikipedia.org/wiki/Password_strength) as possible.
Depending on what the password will be used for, their sources may differ to
provide stronger guarantees for randomness. The random number generation algorithms
can be as simple as using a complex function over a fixed seed or using
environmental events to generate randomness. A list of generators are present at
[Wikipedia - List of random number
generators](https://en.wikipedia.org/wiki/Password_strength).

One of the simplest sources is [`/dev/urandom`](https://linux.die.net/man/4/urandom)
which is a block device serving as an interface to the kernel's random number
generator with a little lesser entropy guarantees.

```bash
$ head -c 16 /dev/urandom | base64
$ head -c 16 /dev/urandom | xxd -ps
$ LC_ALL=C bash -c 'head /dev/urandom | tr -dc a-z0-9 | head -c16'; echo
```

All of these mechanisms use `/dev/urandom` for generating entropy and then
transforming it to a more usable format. For stronger guarantees, `/dev/random`
might also be used.

## We have strong passwords, what next

We have strong passwords now which are not even harder to remember. That's not
good! You would just end up writing the passwords on a Post It note which is even
worse.

The thought is not bad actually, instead of storing it on a Post It note, you
could put it into a file and then protect the file using another password. This
means, you only have to remember one strong password and others are protected
using that password.

Congratulations, we have decoded the basic mechanisms for
* [LastPass](https://lastpass.com)
* [1Password](https://1password.com)
* [DashLane](https://dashlane.com)
* [KeePassX](https://keepassx.org)

Not to forget, the password managers in browsers.

## Password management, the old school way

There are 3 parts to managing the passwords for the modern day,
* Generate a strong password for every login
* Protect all passwords using a strong password like a password manager
* Sync those encrypted passwords across all devices and have a backup for
  disaster management.

Most of the password managers offer the above features, but what if we could do
it ourselves.

Enter [`man pass(1)`](https://linux.die.net/man/1/pass) or
[`zx2c4/pass`](https://git.zx2c4.com/password-store) which is a UNIX-style
password management CLI tool which offers all the above features.

## Using pass for your password workflow

Installation instructions are mostly as-is on the man page for the utility but
there are a few things to keep in mind.

### Prerequisites

The utility requires [`gnupg2`](https://linux.die.net/man/1/gpg2) to be
installed and configured on the host. Generally `GnuPG` can be installed using
the standard package managers.

Once GPG is setup, you will need to generate a key for `pass` to use.
```bash
$ gpg --full-generate-key
```
This would prompt for
* What algorithm to use, you could use `ECC (sign and encrypt)` since we need to
  encrypt the passwords.
* Algorithm parameters, for ECC, the curve to be used, you could use `Curve 25519`
* Expiry for the key

Once this is done, an identity should be provided
* Full Name, eg `Foo Bar`
* Email, eg `foo@bar.com`

This would have the identity `Foo Bar <foo@bar.com>`

> Note: You will need to remember the password used to encrypt the gpg key
> itself. If you loose that, it won't be possible to recover the passwords.

Once the key is generated, list the keys as such
```bash
$ gpg --list-secret-keys
sec   ed25519 2021-10-27 [SC] [expires: 2022-10-27]
      545C94506B84898B3AD27D0682FCD8F82659B1DD
uid           [ultimate] Foo Bar <foo@bar.com>
ssb   cv25519 2021-10-27 [E] [expires: 2022-10-27]
```

Install an [`xclip`](https://linux.die.net/man/1/xclip),
[`pbcopy`](https://www.unix.com/man-page/osx/1/pbcopy/) or an equivalent tool
for clipboard manipulation.

### Installation and initial setup

Installation for `pass` is again fairly simple and is available on all major
platforms using their default package managers. Refer to the [`official
documentation`](https://www.passwordstore.org/) for the steps.

To initialise the pass utility
```bash
$ pass init "your-gpg-key-id"
```
This step initialises pass and tells it which key to use when encrypting or
decrypting your secrets. For example, we want to use the key generated above, we
would have

```bash
$ pass init 545C94506B84898B3AD27D0682FCD8F82659B1DD
```
This should create a file `~/.password-store/.gpg-id` with the keyId as the
contents.

At this point, `pass` is ready to be used. But before that, we need to address
the third requirement.

### Syncing passwords

In order to sync the passwords or have access to the passwords on a different
device, you will need to perform the following actions.

According to the documentation, we would be using `github` as our storage
backend as `pass` uses `git` to manage the password history which makes it
easier to rollback and keep track of when were passwords altered or added.

```bash
$ pass git init
$ pass git remote add origin https://github.com/foo/bar
```
This would convert the `~/.password-store` as a git repository pointing to the
github URL `https://github.com/foo/bar.git`. Any other git backend also can be
used as it delegates the source control to the `git`.

You will want to pull before adding a password and push after adding a password
in order to avoid conflicts.
```bash
$ pass git pull
$ # update passwords
$ pass git push
```
This will push all the changes to the git repository. Make sure to have the git
repo as private.

You have the encrypted passwords, now you need the gpg keys that were used to
encrypt it.
```bash
$ gpg --export-secret-key 545C94506B84898B3AD27D0682FCD8F82659B1DD > pass.asc
```

This will export your key and will be encrypted with the creation password so it
can be hosted on a non-secure channel, either the same git repo or any other
location.

## We are locked and loaded

Once all these steps are done, we are good to store passwords and take control
of our own security.

```bash
$ pass git pull

# Insert password
$ pass insert example.com/username

# Get a password, make sure to use with a `-c` flag to write to clipboard rather
# than stdout
$ pass -c example.com/username

# edit passwords
$ pass edit example.com/username

# delete passwords
$ pass rm example.com/username

# copy passwords to a different folder
$ pass cp exmaple.com/username foobar.com/username

# move passwords to a different folder
$ pass mv example.com/username example.com/newusername
$ pass mv example.com/username personal/example.com/username

$ pass git push
```

Refer to the man page for `pass` for more detailed usage examples. The man page
has a convention of storing the password as `{{ website }}/{{ username }}` but
you are free to organize it as you want.

## Setting up pass on a different machine

The above procedure sets up the frame to use `pass` on a different machine.
Actual steps to do it are

### Setup GnuPG

Download the exported secret key `pass.asc` on the machine and then import the
key using.

```bash
$ gpg --import pass.asc
```
### Setup pass

The `pass` utility needs to be installed on the new host and then setup using
the same gpg key id as found in `~/.password-store/.gpg-id`. Point the git repo
to the same git repo and pull.

```bash
$ pass init "same-gpg-key-id"
$ pass git remote add origin https://github.com/foo/bar
$ pass git pull origin master
```

And that's it!

## Getting to the guts

> Note: I went towards understanding the utility backwards without reading the
> implementation and trying to reverse-engineer the utility, so please bear with me.

### Manually reading
If you inspect the folder `~/.password-store`, you will see a filesystem
structure same as the way passwords were stored and how `pass list` shows.

```bash
$ file ~/.password-store/example.com/username.gpg
```

Since it's a wrapper over GPG, it can be decrypted using `gpg` itself,
```bash
$ gpg --decrypt ~/.password-store/example.com/username.gpg
```
This should give out the correct password cleanly.

### Manually writing
If we can read through the passwords using the standard `gpg` utility, we should
be able to write as well. And that's true!

```bash
$ echo "password" > /tmp/pass.txt
$ gpg --encrypt -o ~/.password-store/example.com/newusername.gpg -r "Foo Bar <foo@bar.com" /tmp/pass.txt
$ pass example.com/newusername
```
Note that we need to supply the identity of the key here which can again be
found using the `gpg --list-secret-keys` command.

## Where I lost 2 hours of my life

This would all work easily in the regular world. But... but but but, when have
versions spared us. GnuPG is no exception to this.

I had initialised the passwords using
```bash
$ gpg --version
gpg (GnuPG) 2.3.3
libgcrypt 1.9.4
```
And was trying to decrypt on a linux (Ubuntu) machine. Everything worked, except
the decryption of existing passwords. `gpg` would not throw any error, even the
exit code was 0.

Initially I assumed that it wasn't writing the password on stdout which would be
a nice thing to do, but that too wasn't the case. I tried using the `-v` switch
and got some more information

```
gpg: public key is XXXX
gpg: using subkey XXXX instead of primary key XXXX
```
And that's it, nothing more.

Tried using the `-vvv` switch to get more information, I would get an error
message

```
gpg: public key is XXXX
gpg: using subkey XXXX instead of public key XXXX
gpg: public key encrypted data; good DEK

:unknown package: type 20, length 95
<Some hex data>
```

And then nothing!

### The breakthrough

In the process of hunting for this error, I came across the issue
[`mozilla/sops #896`](https://github.com/mozilla/sops/issues/896) which seemed
to have a similar issue and had mentioned incompatibility between v2.3.x and
v2.2.x.

The version information for the linux system,
```bash
$ gpg --version
gpg (GnuPG) 2.2.4
libgcrypt 1.8.1
```
Bingo! Now time to test the above issue. Docker to the rescue,

I pulled the image [`vladgh/gpg`](https://hub.docker.com/r/vladgh/gpg) with a
version 2.3 on the linux system, followed the instructions on the page,
replaced the `pass` command with the following in `~/.bash_profile`,

```bash
pass() {
    docker run --rm --it \
        -v /home/$USER/.gnupg:/root/.gnupg \
        -v /home/$USER/.password-store/:/root/.password-store/ \
        -e GPG_TTY=/dev/console vladgh/gpg -d "/root/.password-store/$1.gpg"
}
```

And, voila, it works almost as good as pass but only for decryption!
The only downside is that it won't honor the GPG password
cache and will ask for the GPG key's password.

## Transfer my passwords

Now that we have a way to manually push passwords into pass, we could now insert
passwords from other stores. For example, I was trying to import logins from my
Firefox browsers which has a convenient way to export all logins into a CSV
file. That CSV file can be cleaned and then a simple export `gpg` command should
work.

```sh
insert_password() {
    website=$1;shift;
    username=$1;shift;
    password=$1;shift;

    echo $password > /tmp/pass.txt
    folder="~/.password-store/$website"
    mkdir -p $folder
    output="$folder/$username.gpg"

    rm -f $output
    gpg --encrypt -o $output \
        -r "Foo bar <foo@bar.com>" \
        /tmp/pass.txt
}
```
This function can be called from within a script as
```bash
insert_password "example.com" "username" "password"
```
And would serve as a batch update.

## Community plugins

`pass` is a really extensible tool and allows users to write extensions for it
to add more functionality. The repo
[`tijn/awesome-password-store`](https://github.com/tijn/awesome-password-store)
has a list of community built extensions which provide some interesting
functionality, including the one for bulk importing, GUI interfaces, browser
plugins and much more.

## Note to the readers

In case something is very obvious about GPG that I didn't understand which lead
me to run around like a crazy noob, please feel free to use the reddit link. This
was more of a reverse engineering style of journey without reading the source or
much about GPG.

Discussion link: [/r/cybersecurity passwords over GnuPG](https://www.reddit.com/r/cybersecurity/comments/qhjfqv/passwords_over_gnupg/)
