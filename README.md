# Installing hitch on Ubuntu Trusty (14.04)

[Hitch](https://hitch-tls.org/) is a libev-based high performance SSL/TLS proxy by Varnish Software.
It can be used to terminate SSL in front of Varnish.
However, there is no precompiled binaries for `hitch` on Ubuntu Trusty (14.04), and we need to compile them from source.

## Installing from source

### Install `OpenSSL`

`OpenSSL 1.0.1f` is distributed with Ubuntu Trusty, but there are bugs and vulnerabilities with this version. Compile the latest version instead.

Download the latest [OpenSSL](https://www.openssl.org/source/) binaries, and installed as shared library:

    sudo apt-get install make
    wget https://www.openssl.org/source/openssl-1.0.2l.tar.gz
    tar -xzvf openssl-1.0.2l.tar.gz
    cd openssl-1.0.2l
    ./config shared
    make
    sudo make install

To verify OpenSSL is installed correctly:

    /usr/local/ssl/bin/openssl version -v

### Install `hitch`:

We need to link the OpenSSL to the version we installed in `/usr/local/ssl/`.

    sudo apt-get install libev-dev automake python-docutils flex bison pkg-config
    wget https://hitch-tls.org/source/hitch-1.4.6.tar.gz
    tar -xzvf hitch-1.4.6.tar.gz
    cd hitch-1.4.6
    ./bootstrap
    env SSL_CFLAGS="-I/usr/local/ssl/include" \ 
          SSL_LIBS="-L/usr/local/ssl/lib -Wl,-R/usr/local/ssl/lib -lssl" \
          CRYPTO_CFLAGS="-I/usr/local/ssl/include" \
          CRYPTO_LIBS="-L/usr/local/ssl/lib -Wl,-R/usr/local/ssl/lib -lcrypto" ./configure
    make
    sudo make install

This will install Hitch to `/usr/local/`.

To check whether the OpenSSL is linked correctly, you can verify the dependencies:

    ldd src/hitch

Make sure that OpenSSL is linked to `/usr/local/ssl/lib/libssl.so.1.0.0`.

## Configure Varnish
We want Varnish to forward all challenge requests to Acmetool, and we are going to create a request matching rule in VCL that will ensure this forwarding happens.

Create `/etc/varnish/acmetool.vcl` with the following:
  
    # Forward challenge-requests to acmetool, which will listen to port 402
    # when issuing lets encrypt requests

    backend acmetool {
      .host = "127.0.0.1";
      .port = "402";
    }

    sub vcl_recv {
      if (req.url ~ "^/.well-known/acme-challenge/") {
        set req.backend = acmetool;
        return(pass);
      }
    }

Then we need to include this in our main VCL `/etc/varnish/default.vcl`, and add the VCL below your backend definitions: 

    include "/etc/varnish/acmetool.vcl";

Check whether VCL is correct:

    sudo varnishd -C -f /etc/varnish/default.vcl

And reload the VCL:

    sudo /etc/init.d/varnish reload

## Install Acmetool

[Acmetool](https://hlandau.github.io/acme/) is published in a PPA, so we will add this and then install the package:

    sudo add-apt-repository ppa:hlandau/rhea
    sudo apt-get update
    sudo apt-get install acmetool

## Acquire the certificate

Now we will use Acmetool to acquire a certificate.

Now we have everything in place and we run the Acmetool quickstart process. It should detect that we are using Hitch and automatically set up a hook that will generate Hitch-compatible certificate-packages from certificate requests.

    sudo acmetool quickstart

Answer the prompts like this to enable live certificates authenticated through challenge requests proxied through Varnish.

    ------------------------- Select ACME Server -----------------------
    1) Let's Encrypt (Live) - I want live certificates

    ----------------- Select Challenge Conveyance Method ---------------
    2) PROXY - I'll proxy challenge requests to an HTTP server

Review and (hopefully) accept the letsencrypt.org Terms of Service, and enter your email address.

    -------------------- Install HAProxy/Hitch hooks? ------------------
    Yes) Do you want to install the HAProxy/Hitch notification hook?

    -------------------- Install auto-renewal cronjob? -----------------
    Yes) Would you like to install a cronjob to renew certificates automatically? This is recommended.

Before we continue to requesting our certificate we need to generate a Diffie-Hellman group file (aka dhparams), used for perfect forward secrecy.

    sudo openssl dhparam -out /var/lib/acme/conf/dhparams 2048

Now we can finally get our certificate (replace `example.com` with your FQDN):

    sudo acmetool want example.com

## Hitch Configuraiton

 Create the file `/etc/hitch/hitch.conf`

    ## Basic hitch config for use with Varnish and Acmetool

    # Listening
    frontend = "[*]:443"
    ciphers  = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"

    # Send traffic to the Varnish backend using the PROXY protocol
    backend  = "[::1]:80"

    # List of PEM files, each with key, certificates and dhparams
    # Replace example.com with your FQDN.
    pem-file = "/var/lib/acme/live/example.com/haproxy"

Create the user and group to run `hitch`:

    sudo addgroup --system hitch
    sudo adduser --system --disabled-login --gecos "Hitch TLS Unwrapper" --home /nonexistent --no-create-home --ingroup hitch --shell /bin/false hitch

Start Hitch with the new configuration:

    sudo hitch -u hitch -g hitch --config=/etc/hitch/hitch.conf

Remember to open the https port on the firewall:

    utw allow https

## Init scripts

Download the init.d script from: 
https://github.com/wjtan/hitch-ubuntu-trusty/blob/master/hitch

    sudo chmod 755 /etc/init.d/hitch
    sudo chown root:root /etc/init.d/hitch

Add and enable the script:

    sudo update-rc.d hitch defaults
    sudo update-rc.d hitch enable

Start init script:

    sudo /etc/init.d/hitch start

Initialization scripts to start hitch automatically can also be found in the [wiki](https://github.com/varnish/hitch/wiki).

