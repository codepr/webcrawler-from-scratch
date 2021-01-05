# Go installation and project init

Go installation is pretty simple and straight-forward,
[https://golang.org/doc/install](https://golang.org/doc/install) explain each
simple step to follow for every host platform. Being a Linux junky I'll follow
that one:

1. Download a `.tar.gz`

```
wget https://golang.org/dl/go1.15.5.linux-amd64.tar.gz
```

2. Download the archive and extract it into `/usr/local`, creating a Go tree in `/usr/local/go`.

```
sudo tar -C /usr/local -xzf go1.15.5.linux-amd64.tar.gz
```

3. Add `/usr/local/go/bin` to the PATH environment variable.
     You can do this by adding the following line to your `$HOME/.profile` or `/etc/profile` (for a system-wide installation):

     **Note:** Changes made to a profile file may not apply until the next time
     you log into your computer. To apply the changes immediately, just run the
     shell commands directly or execute them from the profile using a command
     such as source `$HOME/.profile`.

```
 export PATH=$PATH:/usr/local/go/bin
```

4. Verify that you've installed Go by opening a command prompt and typing the following command:

```
go version
```

After the installation is complete and successful we just need to create the
project folder

```
mkdir webcrawler && cd webcrawler
```

and init it

```
go mod init webcrawler
```
