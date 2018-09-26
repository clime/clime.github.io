---
layout: post
title: Starting your own distro
lang: en
---

Prolog
------

This is an update on this blog post <https://clime.github.io/2017/05/20/DistGit-1.0.html> and mainly on its hands-on part.

There have been a continuous effort in the area of linux distribution maintenance to make its tooling reusable and accessible
for everyone. Finally, this note is presenting the results of this effort to you. It will give you an idea
how big distros like Fedora or RHEL are maintained and it will also show you a way to start and maintain your own linux
distribution.

Okay, show me what you've got
-----------------------------

The following tutorial will be run with `dist-git 1.8-1` and `rpkg 2.6-1`. You may find packages of those or higher versions
already in official Fedora and EPEL repositories but we will be using Fedora coprs [clime/rpkg-util](https://copr.fedorainfracloud.org/coprs/clime/rpkg-util/)
and [clime/dist-git](https://copr.fedorainfracloud.org/coprs/clime/dist-git/) to ensure we have the needed up-to-date versions.

*Note: On EPEL, we will also use updated `git 1.8.3.1-19_git_c` from [clime/rpkg-util](https://copr.fedorainfracloud.org/coprs/clime/rpkg-util/).*

If you use vagrant and you want to skip the setup part below, you can use this [Vagrantfile](https://clime.cz/dist-git/Vagrantfile) for Fedora-28.
You can then immediately jump <a href="#setup-end">here</a>.

Otherwise, let's begin!

Normally, you would use a server machine and a client machine but for simple playing around, we can just use localhost
to substitute for both. We recommend to create an isolated container or a virtual machine to carry out the tutorial.

First, install the dist-git package:

    # dnf install dnf-plugins-core
    # dnf copr enable clime/dist-git
    # dnf install dist-git

on CentOS7 or RHEL7 with EPEL7 (`yum install epel-release`) enabled:

    # yum install yum-plugin-copr
    # yum copr enable clime/dist-git
    # yum install dist-git

Next step is to setup Apache for uploading source tarballs.

There is `/etc/httpd/conf.d/dist-git/lookaside-upload.conf.example` provided by the dist-git package
itself for ssl uploading with authentication by client certificates but we will use something much more
simple. Put the following lines into `/etc/httpd/conf.d/dist-git/lookaside-upload.conf`:

    <VirtualHost _default_:80>
        # This alias must come before the /repo/ one to avoid being overridden.
        ScriptAlias /repo/pkgs/upload.cgi /var/lib/dist-git/web/upload.cgi

        Alias /repo/ /var/lib/dist-git/cache/lookaside/

        LogLevel trace8

        # provide username manually to upload.cgi
        SetEnv SSL_CLIENT_S_DN_CN joe

        <Location /repo/pkgs/upload.cgi>
            Options +ExecCGI
            Require all granted
        </Location>
    </VirtualHost>

Now you can start the httpd server:

    # systemctl start httpd

If you hit problems with localhost ssl certs missing on httpd start, move
`/etc/httpd/conf.d/ssl.conf` to `/etc/httpd/conf.d/ssl.conf.off` and try again.

We will now create two users to carry out all the unprivileged tutorial actions.

User `joe` will be responsible for all client operations (i.e. cloning/pushing) and
user `admin` will be responsible for all server operations (i.e. setting up a new
package repo/chilling out). Both `joe` and `admin` need to belong to `packager`
group that got created on installation of the dist-git package.

    # useradd admin -G packager
    # useradd joe -G packager

There is very few things missing to set up the server part at this point. First,
start ``dist-git.socket`` service so that `git://` protocol works for anonymous read-only
access (we shall use it later):

<pre>
# systemctl start dist-git.socket
# systemctl status dist-git.socket  <span class="comment"># state should be "active (listening)"</span>
</pre>

Now we will make sure sshd is up and running which we will be used for authorized
Git read/write access.

To install ssh server on Fedora, run:

    # dnf install openssh-server

To install it on EPEL, run:

    # yum install openssh-server

Then finally on both distros, you can run:

<pre>
# systemctl start sshd
# systemctl status sshd  <span class="comment"># state should be "active (running)"</span>
</pre>

Also, let's configure public key access to localhost for user joe:

<pre>
# su joe
joe@localhost / $ cd
joe@localhost ~ $ ssh-keygen  <span class="comment"># press enter on everything</span>
joe@localhost ~ $ cat .ssh/id_rsa.pub >> .ssh/authorized_keys
joe@localhost ~ $ chmod 600 .ssh/authorized_keys
joe@localhost ~ $ ssh localhost  <span class="comment"># on Fedora, you might need to do rm /run/nologin as root</span>
</pre>

Now for the client part, install the `rpkg` package. We will use the latest package version from
[copr.fedoraproject.org/clime/rpkg-util](https://copr.fedorainfracloud.org/coprs/clime/rpkg-util/) project.

On Fedora, you can invoke:

    # dnf copr enable clime/rpkg-util
    # dnf install rpkg

On EPEL, you can do:

<pre>
# yum copr enable clime/rpkg-util
# yum install rpkg
# yum install git  <span class="comment"># to upgrade git from the enabled copr repo</span>
</pre>

Put the following configuration into `/etc/rpkg.conf` (by replacing the default content):

    [rpkg]
    preprocess_spec = True

    # auto-packing is deprecated:
    auto_pack = False

    base_output_path = /tmp/rpkg

    [git]
    lookaside = http://localhost/repo/pkgs/%(ns1)s/%(name)s/%(filename)s/%(hashtype)s/%(hash)s/%(filename)s
    lookaside_cgi = http://localhost/repo/pkgs/upload.cgi
    gitbaseurl = ssh://%(user)s@localhost/var/lib/dist-git/git/%(module)s
    anongiturl = git://localhost/%(module)s

<span id="setup-end"></span>That's it! Let's create our first DistGit repository.

<pre>
# su admin
admin@localhost / $ /usr/share/dist-git/setup_git_package package  <span class="comment"># creates Git repo on the server</span>
Generating initial grok manifest...
Done.
admin@localhost / $ ls /var/lib/dist-git/git/rpms
package.git
</pre>

Now you can already interact with the created DistGit repository by using rpkg:
    
<pre>
admin@localhost / $ exit
# su joe
joe@localhost / $ cd
joe@localhost ~ $ rpkg clone package  <span class="comment"># clones the package.git repo</span>
joe@localhost ~ $ cd package
joe@localhost package $ ls
sources
</pre>

Now we are in our local cloned Git repository. Let's initialize it with some public source rpm:

<pre>
joe@localhost package $ curl https://clime.cz/prunerepo-1.13-1.fc28.src.rpm -o /tmp/prunerepo-1.13-1.fc28.src.rpm
joe@localhost package $ rpkg import /tmp/prunerepo-1.13-1.fc28.src.rpm  <span class="comment"># unpack src.rpm, upload tarball into dist-git's lookaside and modify local repo accordingly</span>
joe@localhost package $ git status  <span class="comment"># display what has been changed in the local git repo</span>
On branch master
Your branch is up-to-date with 'origin/master'.

Changes to be committed:
  (use "git reset HEAD <span><</span>file<span>></span>..." to unstage)

        modified:   .gitignore
        new file:   prunerepo.spec
        modified:   sources

joe@localhost package $ cat sources  <span class="comment"># let's display the pointer to the lookaside cache for the uploaded tarball</span>
SHA512 (prunerepo-1.13.tar.gz) = 25c3f6e42f390e4e2215f0f24fea4a0482ee910ce7fa129c8d91c33bf350d31c564796721437a053ad34bdddb67c36cbb8130b5e54c5bf6af9d68bed0e983244

joe@localhost package $ git config --global user.email "joe@localhost"  <span class="comment"># set user git commit info</span>
joe@localhost package $ git config --global user.name "joe"

joe@localhost package $ git commit -m "DistGit test update" -a  <span class="comment"># commit changes to the local Git repo</span>
joe@localhost package $ git rev-parse master  <span class="comment"># show commit hash, output will differ for you</span>
d8d68e0d8e47455ca686516b45a65e37d752fbbd
joe@localhost package $ rpkg srpm  <span class="comment"># build srpm just to test things out</span>
joe@localhost package $ rpkg push  <span class="comment"># push local git changes to DistGit</span>
Wrote: /tmp/rpkg/prunerepo-1-qzygm0hc/prunerepo.spec
Wrote: /tmp/rpkg/prunerepo-1-qzygm0hc/prunerepo-1.13-1.fc28.src.rpm
joe@localhost package $ rpkg build  <span class="comment"># build package in Copr BuildSystem, this needs copr-cli tool to be installed</span>
...
</pre>

You can also verify that the changes got into the DistGit server:

    joe@localhost package $ cat /var/lib/dist-git/git/rpms/package.git/refs/heads/master
    d8d68e0d8e47455ca686516b45a65e37d752fbbd
    joe@localhost package $ ls /var/lib/dist-git/cache/lookaside/pkgs/rpms/package/prunerepo-1.13.tar.gz/sha512/25c3f6e42f390e4e2215f0f24fea4a0482ee910ce7fa129c8d91c33bf350d31c564796721437a053ad34bdddb67c36cbb8130b5e54c5bf6af9d68bed0e983244/prunerepo-1.13.tar.gz

And you can, of course, now clone the repo and start doing something from scratch.
Let's do it with `-a` switch, which uses the `git://` scheme and the read-only git-smart-http Git backend.

<pre>
joe@localhost package $ cd
joe@localhost ~ $ rpkg clone -a package package-copy
joe@localhost package-copy $ cd package-copy
joe@localhost package-copy $ ls
prunerepo.spec  sources
joe@localhost package-copy $ rpkg sources  <span class="comment"># fetch the tarball</span>
Downloading prunerepo-1.13.tar.gz from lookaside cache at localhost
######################################################################## 100.0%
joe@localhost package-copy $ ls
prunerepo-1.13.tar.gz  prunerepo.spec  sources
joe@localhost package-copy $ rpkg srpm  <span class="comment"># build srpm again just to see that it works</span>
Wrote: /tmp/rpkg/prunerepo-2-_w0wu16l/prunerepo.spec
Wrote: /tmp/rpkg/prunerepo-2-_w0wu16l/prunerepo-1.13-1.fc27.src.rpm
</pre>

Pretty cool, right? You can basically start your own linux distribution from this
very basic initial setup.

Needless to say, so far we have show-cased work with traditional packed sources only (spec + patches + tarballs)
and this is how Fedora, CentOS, RHEL, Mageia and other distros work.

What is quite interesting about this setup (DistGit+rpkg) is that you can have unpacked sources repos
(spec + raw source files) in DistGit as well and that's thanks to support for this in the rpkg utility.

Let's take <https://pagure.io/hello_rpkg> project, which is raw sources with spec, and import
it to our setup here. 

<pre>
<span class="comment"># Here we are no longer mentioning commands for switching between users and dirs.</span>

admin@localhost / $ /usr/share/dist-git/setup_git_package hello_rpkg

joe@localhost ~ $ rpkg clone hello_rpkg
joe@localhost ~ $ cd hello_rpkg
joe@localhost hello_rpkg $ git pull https://pagure.io/hello_rpkg --allow-unrelated-histories  <span class="comment"># on EPEL, omit --allow-unrelated-histories switch</span>
joe@localhost hello_rpkg $ ls
Makefile  README.md  hello_rpkg.spec.rpkg  main.c  sources
</pre>

So we have imported code and history from <https://pagure.io/hello_rpkg> project. There is `sources` file in addition
to the content at <https://pagure.io/hello_rpkg>, which was created by `/usr/share/dist-git/setup_git_package` script.
We may remove it as we won't probably be using lookaside cache for this particular project.

    joe@localhost hello_rpkg $ rm sources
    joe@localhost hello_rpkg $ git commit -a -m 'remove unneeded sources file'

**Note:** In the current upstream version of dist-git at <https://github.com/release-engineering/dist-git>,
the empty 'sources' file is no longer being pregenerated.

And let's push:

    joe@localhost hello_rpkg $ rpkg push

to get the code and history import finished.

Now it is time a play around with the code a little bit. So let's again try to generate an srpm, this time from unpacked sources:

    joe@localhost hello_rpkg $ rpkg srpm
    git_dir_pack: packing path /home/joe/hello_rpkg
    git_dir_pack: Wrote: /tmp/rpkg/hello_rpkg-1-2oz9vvwk/hello_rpkg-0.0.git.8.1a2615b.tar.gz
    Wrote: /tmp/rpkg/hello_rpkg-1-2oz9vvwk/hello_rpkg.spec
    Wrote: /tmp/rpkg/hello_rpkg-1-2oz9vvwk/hello_rpkg-0.0.git.8.1a2615b-1.fc28.src.rpm

You can see that it works as well as it worked for the packed case (spec+patches+tarballs). How is that
even possible? You will find out when you closer examine the `hello_rpkg.spec.rpkg` file, which is an rpkg
spec file template. Particularly, let's examine the line defining an rpm source (`'Source:'`), which is
usually a name of tarball stored in the lookaside cache:

<pre>
joe@localhost hello_rpkg $ grep 'Source:' hello_rpkg.spec.rpkg
Source:     {% raw %}{{{ git_dir_pack }}}{% endraw %}
</pre>

`{% raw %}{{{ git_dir_pack }}}{% endraw %}` is a special rpkg macro, which tells rpkg that the
tarball should be dynamically generated from the Git checked-out content. That generated tarball
will be then used to build the final srpm. This is different from the standard procedure where
the tarball is statically present next to the spec file (even though just as a link to the lookaside
until you download it) and can be just used to build an srpm.

This feature of `rpkg` utility enables you to work with the sources in their unpacked form and
only pack them when you need to build them. With this feature, instead of adding patch files
and `Patch:` directives into the spec file, you could just commit the changes without needing
to generate patch files at all.

That's it. This tutorial should give you the basic gist of how it works under the hood in Fedora and
similarly, in other rpm distros. All those distros still use just the traditional spec+patches+tarballs
approach. So if you use the setup presented here, you are going to be ahead of them as far as Git package
maintenance goes.

Anything else?
--------------

The DistGit upstream is hosted at <https://github.com/release-engineering/dist-git>.

The rpkg-util upstream is hosted at <https://pagure.io/rpkg-util>.

Please, send us patches and requests there.
