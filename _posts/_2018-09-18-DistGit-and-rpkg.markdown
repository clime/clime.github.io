DistGit and rpkg
================

Prolog
------

This is an update on this blog post https://clime.github.io/2017/05/20/DistGit-1.0.html and on the hands-on part in particular.
There have been many advancements in this area lately and this quick note should help you to setup your own DistGit instance
and show you how to easily handle it with rpkg utility. rpkg and DistGit combination is useful if you wish to maintain
your own set of rpm packages. Originally, it was possible to only store tarballs+patches in DistGit but the latest upgrades in
rpkg-util allows users to store unpacked (raw) sources in the DistGit repositories as well and the lookaside cache for storing
tarballs or other large files becomes an optional feature. That is a pretty great advancement in the packaging world because
packager and upstream developer workflows come much closer together.

Okay, I want to try this DistGit and rpkg combo out. How?
---------------------------------------------------------

Normally, you would need a server machine and a client machine but for playing around, we will just use localhost for both.
We recommend to do the following procedure in an isolated container. We will asume that all the unprivileged operations are
done by user fred.

First, install the dist-git package on the DistGit server (=localhost):

    $ sudo dnf install dist-git

or on CentOS7 or RHEL7 with EPEL7 enabled:

    $ sudo yum install dist-git

You can now setup Apache on the server (=localhost) for uploading source tarballs. In case you would like to store
unpacked sources on DistGit only, you don't even need to do that. DistGit then essentially becomes just a normal Git
server with some configuration already in place.

So you can just skip this part if you don't need the lookaside cache...

There is ``/etc/httpd/conf.d/dist-git/lookaside-upload.conf.example`` provided by the package
for ssl uploading with client certificates for authentication but we will use something
more simple for the demonstration. Put the following into ``/etc/httpd/conf.d/dist-git/lookaside-upload.conf``:

    <VirtualHost _default_:80>
        # This alias must come before the /repo/ one to avoid being overridden.
        ScriptAlias /repo/pkgs/upload.cgi /var/lib/dist-git/web/upload.cgi

        Alias /repo/ /var/lib/dist-git/cache/lookaside/

        LogLevel trace8

        # provide username manually to upload.cgi
        SetEnv SSL_CLIENT_S_DN_CN fred

        <Location /repo/pkgs/upload.cgi>
            Options +ExecCGI
            Require all granted
        </Location>

    </VirtualHost>

Now you can start the httpd server:

    $ sudo systemctl start httpd

If you hit problems with localhost ssl certs missing, move `/etc/httpd/conf.d/ssl.conf`
into `/etc/httpd/conf.d/ssl.conf.off`.

Now make sure fred belongs to `packager` group:

    $ sudo usermod fred -G packager

This will allow you to upload tarballs to DistGit's lookaside cache as well as push changes.
Just to enable uploading, you could also go to `/etc/dist-git/dist-git.conf` and set
`disable_group_check = True` in `[upload]` section. 

That's it for the lookaside cache configuration. For the server-side configuration, there is very few
things missing. First, start dist-git.socket service so that `git://` protocol works for anonymous
read-only access:

    $ sudo systemctl start dist-git.socket

And also let's configure public key access to localhost for user fred:

    $ ssh-keygen
    $ cat .ssh/id_rsa.pub >> .ssh/authorized_keys
    $ chmod 600 .ssh/authorized_keys

Now for the client part, install the `rpkg` package. On Fedora, you can just invoke:

    $ sudo dnf install rpkg

On EPEL6 or EPEL7, you can install the rpkg package from a copr repository:

    $ sudo yum install yum-plugin-copr
    $ sudo yum copr enable clime/rpkg-util
    $ sudo yum install rpkg

Now put the following configuration into ``/etc/rpkg.conf``:

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

Let's create our first DistGit repository. This needs to be done server-side:

    root@localhost $ /usr/share/dist-git/setup_git_package foo # creates Git repo on the server
    root@localhost $ ls /var/lib/dist-git/git/rpms
    foo.git

You don't need to use `root` user for the repository creation at the server.
Any user in the `packager` group is suitable.

Now you can already interact with the created DistGit repository by using rpkg:

    $ rpkg clone package # clones remote foo.git repo
    $ cd package

Now we are in our local Git repository. Let's initialize it with some public source rpm:

    $ curl https://clime.cz/prunerepo-1.13-1.fc28.src.rpm -o /tmp/prunerepo-1.13-1.fc28.src.rpm
    $ rpkg import /tmp/prunerepo-1.13-1.fc28.src.rpm   # unpack src.rpm, upload tarballs dist-git's lookaside and modify local repo accordingly
    $ git status                                       # display what has been changed in the local git repo
    On branch master
    Your branch is up-to-date with 'origin/master'.

    Changes to be committed:
      (use "git reset HEAD <file>..." to unstage)

            modified:   .gitignore
            new file:   prunerepo.spec
            modified:   sources

    $ cat sources                                      # let's display the pointer to the lookaside cache for the uploaded tarball
    SHA512 (prunerepo-1.13.tar.gz) = 25c3f6e42f390e4e2215f0f24fea4a0482ee910ce7fa129c8d91c33bf350d31c564796721437a053ad34bdddb67c36cbb8130b5e54c5bf6af9d68bed0e983244

    $ git config --global user.email "fred@localhost"  # set user git commit info
    $ git config --global user.name "fred"

    $ git commit -m "DistGit test update" -a           # commit changes to the local Git repo
    $ git rev-parse master                             # show commit hash
    d8d68e0d8e47455ca686516b45a65e37d752fbbd
    $ rpkg push                                        # push local git changes to DistGit, rpkg push invokes git push --follow-tags 
    $ rpkg srpm                                        # build srpm just to test things out
    Wrote: /tmp/rpkg/prunerepo-1-qzygm0hc/prunerepo.spec
    Wrote: /tmp/rpkg/prunerepo-1-qzygm0hc/prunerepo-1.13-1.fc27.src.rpm
    $ rpkg build                                       # build package in Copr BuildSystem, this needs copr-cli tool to be installed
    ...

You can also verify that the changes got into the DistGit server:

    $ cat /var/lib/dist-git/git/rpms/package.git/refs/heads/master
    d8d68e0d8e47455ca686516b45a65e37d752fbbd
    $ ls /var/lib/dist-git/cache/lookaside/pkgs/rpms/package/prunerepo-1.13.tar.gz/sha512/25c3f6e42f390e4e2215f0f24fea4a0482ee910ce7fa129c8d91c33bf350d31c564796721437a053ad34bdddb67c36cbb8130b5e54c5bf6af9d68bed0e983244/prunerepo-1.13.tar.gz

And you can, of course, now clone the repo and start from scratch. Let's do it with -a switch,
which uses the git:// scheme and read-only git-smart-http Git backend.

    $ rpkg clone -a package package-copy 
    $ cd package-copy
    $ ls
    prunerepo.spec  sources
    $ rpkg sources                            # fetch the tarball
    Downloading prunerepo-1.13.tar.gz from lookaside cache at localhost
    ######################################################################## 100.0%
    $ ls
    prunerepo-1.13.tar.gz  prunerepo.spec  sources
    $ rpkg srpm                               # build srpm again just to see that it works
    Wrote: /tmp/rpkg/prunerepo-2-_w0wu16l/prunerepo.spec
    Wrote: /tmp/rpkg/prunerepo-2-_w0wu16l/prunerepo-1.13-1.fc27.src.rpm

Pretty cool, right? You basically can start your own linux distribution from this
very basic initial setup.

Needless to say, so far we have show-cased work with traditional packed sources only (spec+patches+tarballs)
and this is how Fedora, CentOS, RHEL, Mageia, ... distros work. But what is quite unique on this setup now
(DistGit+rpkg) is that you can have unpacked sources repos as well and that's thanks to support for this
in the rpkg utility, which brings a new feature called "rpkg macros". Let's take https://pagure.io/rpkg-util
project itself, which is raw sources with spec, and port it to our setup here.

FIXME:

    # /usr/share/dist-git/setup_git_package rpkg-util
    $ git clone https://pagure.io/rpkg-util
    $ cd rpkg-util
    $ git remote set-url origin ssh://fred@localhost/var/lib/dist-git/git/rpms/rpkg-util
    $ git pull --rebase
    $ rpkg push

So we have finished the migration.


Anything else?
--------------

The DistGit upstream is hosted at <https://github.com/release-engineering/dist-git>.

The rpkg-util upstream is hosted at <https://pagure.io/rpkg-util>.

Please, send us patches and requests there.
