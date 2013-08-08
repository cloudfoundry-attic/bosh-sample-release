# BOSH Sample Release

This is a sample release repository for [BOSH](https://github.com/cloudfoundry/bosh) that deploys a three tier LAMP application: a wordpress blog which consists of a number of apache servers running php & wordpress, fronted by nginx, and using one mysql database for storage.

This document includes a tutorial on how to create a BOSH release.  If you want to move directly to deploying the sample release skip to the [deployment instructions](#deploy).

## creating a BOSH release

### INDEX

* [getting started](#start)
* [creating a package](#package)
* [creating a job](#job)

### release contents

A BOSH release consists of jobs, packages, source code, and associated release metadata.  Large binary files (aka "blobs") required for packaging can be stored externally in a blobstore and referenced within the release, obviating the need to store them inside the release repository.  Additionally, in order to utilize a BOSH release you must create a deployment manifest.

For a more detailed explanation of BOSH terminology please refer to the [BOSH Reference](http://docs.cloudfoundry.com/docs/running/bosh/reference/).

###  <a id="start"></a>getting started

The first step is to initialize the release:

    $ bosh init release bosh-sample-release

which creates the release scaffolding:

    $ find bosh-sample-release
    bosh-sample-release
    bosh-sample-release/blobs
    bosh-sample-release/config
    bosh-sample-release/config/blobs.yml
    bosh-sample-release/jobs
    bosh-sample-release/packages
    bosh-sample-release/src

For now we are going to concentrate on the contents of the following directories:

* src: this is the source code for the packages
* packages: these are the instructions on how to compile the source into binaries
* jobs: these are the scripts and configuration files required to run the packages

#### organize your cloud deployment into jobs

In order to create a BOSH release you will need to map your existing cloud service into "[jobs](http://docs.cloudfoundry.com/docs/running/bosh/reference/jobs.html)", the fundamental building block of a BOSH deployment.  Essentially this is just creating logical groupings of processes that are organized around providing discrete functionality.  Reviewing your architecture diagram is a good place to start.

In the case of our sample release, we are implementing an architecture with four easily recognizable roles: a web application (1) sitting behind a proxy (2), leveraging a database server (3) for dynamic content and a shared filesystem (4) for static content.

#### identify processes, packages, and dependencies

For each job you will need to identify:

* processes that need to run
* packages that are required for running these processes
* package dependencies (e.g. libraries)

The first job of our release is the wordpress web application, which will be served up by the Apache webserver.  For now we'll work under the assumption that apache will be the only process we need to control for this job.  Now we need to determine what BOSH packages need to be created for apache to serve wordpress.

> It is important to note that when we refer to a package we are referring to [BOSH packages](http://docs.cloudfoundry.com/docs/running/bosh/reference/packages.html).  Do not confuse BOSH packages with "packages" installed by your operating system's package manager.  The OS package manager is generally only used to manage the components of the [BOSH stemcell](http://docs.cloudfoundry.com/docs/running/bosh/components/stemcell.html).

Obviously, our first package is the apache webserver.  The wordpress app is another package.  Wordpress is written in PHP, so we have a dependency on the apache PHP module.  Our database backend is mysql, so our PHP will have a dependency on mysql libraries being available.  Our architecture also uses a shared filesystem, but we don't need to package the NFS client because it is available as part of the stemcell.

For the first job (wordpress) we have identified one process (apache) and four packages.  Fortunately, the remaining jobs (nginx, mysql, nfs server) are fairly simple.  They each essentially run a single process and do not have  dependency chains, leaving us with the following:

jobs

* wordpress (web application)
* nginx (proxy)
* mysql (db)
* nfs server (shared filesystem)

packages

* apache (wordpress)
* wordpress (wordpress)
* php (wordpress)
* mysql (wordpress, mysql)
* nginx (nginx)
* nfs server (nfs server)

### <a id="package"></a>creating a package

Once you have enumerated your jobs and pacakges, the next step is to start building packages.

#### generate the package skeletons

First, run the BOSH command to create the package skeletons

e.g. apache2

    $ cd bosh-sample-release
    $ bosh generate package apache2
    create	packages/apache2
    create	packages/apache2/packaging
    create	packages/apache2/pre_packaging
    create	packages/apache2/spec

    Generated skeleton for `apache2' package in `packages/apache2'

Then all other packages

    $ echo common debian_nfs_server mysql nginx php5 wordpress | xargs -n1 bosh generate package

#### define the package specs

The spec file is a yaml document that lists the names of the required source files and other packages which are compilation dependencies.

The spec file for our apache package is quite simple:

**packages/apache2/spec**

    ---
    name: apache2

    files:
      - apache2/httpd-2.2.25.tar.gz

The spec file for PHP is a bit more interesting, as the apache and mysql packages are compile time dependencies.  Additionally, since the mysql source is separated into server/client components we've broken mysql up into two separate packages (mysql and mysqlclient).

**packages/php5/spec**

    ---
    name: php5

    dependencies:
      - apache2
      - mysqlclient

    files:
      - php5/php-5.3.10.tar.gz

#### add the source code to the release

You'll notice that the location of the source in the package spec is a relative path.  This is because the file can either be a part of the repo (inside the src directory) or stored in the blobstore (synched to blobs directory).  In the case of our sample release we're storing source in the repo:

    $ mkdir src/apache2
    $ cd src/apache2
    $ wget 'http://apache.tradebit.com/pub/httpd/httpd-2.2.25.tar.gz'

    Resolving apache.tradebit.com... 71.6.202.162
    Connecting to apache.tradebit.com|71.6.202.162|:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 7445734 (7.1M) [application/x-gzip]
    Saving to: `httpd-2.2.25.tar.gz'

    100%[======================================>] 7,445,734    798K/s   in 9.7s

TODO: blob example

#### write the compilation script

At compile time BOSH executes the [packaging](http://docs.cloudfoundry.com/docs/running/bosh/reference/packages.html#package-compilation) script.  The source code will be located in the current working directory and can also be referenced by the environment variable BOSH\_COMPILE\_TARGET.  Your script should install the compiled package underneath the BOSH\_INSTALL\_TARGET.  If you are installing software that uses GNU autoconf your packaging script can often be as simple as untar-ing your source, running configure with the prefix set to BOSH\_INSTALL\_TARGET, followed by a "make install".

For example, our apache packaging script is a perfect example of a simple configure/install process:

**packages/apache2/packaging**

    cat <<"EOF" >packages/apache2/packaging
    echo "Extracting apache https..."
    tar xzf apache2/httpd-2.2.25.tar.gz

    echo "Building apache https..."
    (
        cd httpd-2.2.25
        ./configure \
        --prefix=${BOSH_INSTALL_TARGET} \
        --enable-so
        make
        make install
    )
    EOF

Due to the compile time dependencies PHP is more complicated:

**packages/php5/packaging**

    cat <<"EOF" >packages/php5/packaging
    echo "Extracting php5..."
    tar xjf php5/php-5.3.27.tar.bz2

    # ./configure needs the location of the apache and mysql packages
    export APACHE2=/var/vcap/packages/apache2

    echo "Building php5..."
    cd php-5.3.27
      ./configure \
      --prefix=${BOSH_INSTALL_TARGET} \
      --with-apxs2=${APACHE2}/bin/apxs \
      --with-config-file-path=/var/vcap/jobs/wordpress/config/php.ini \
      --with-mysql=/var/vcap/packages/mysqlclient
    EOF

>stdout from make is redirected to /dev/null because the output is too large for the [nats message bus](http://docs.cloudfoundry.com/docs/running/bosh/components/messaging.html)

    cat <<EOF >>packages/php5/packaging
    make > /dev/null
    make install > /dev/null
    EOF

>finally, make installs into the apache module directory, we have to manually copy it back into BOSH\_INSTALL\_TARGET otherwise it will be left behind on the compilation vm

    cat <<"EOF" >>packages/php5/packaging
    mkdir -p ${BOSH_INSTALL_TARGET}/modules
    cp ${APACHE2}/modules/libphp5.so ${BOSH_INSTALL_TARGET}/modules
    EOF

### <a id="job"></a>creating a job

Now that all of the packages have been created we can start assembling [jobs](http://docs.cloudfoundry.com/docs/running/bosh/reference/jobs.html).

#### generate the job skeleton

First, run the BOSH command to create the job skeleton:

    $ cd bosh-sample-release/
    $ bosh generate job wordpress
    create	jobs/wordpress
    create	jobs/wordpress/templates
    create	jobs/wordpress/spec
    create	jobs/wordpress/monit

    Generated skeleton for `wordpress' job in `jobs/wordpress'

#### contents of the job spec

The spec file is a yaml document that defines

* required packages
* paths to the configuration templates
* job properties (plus descriptions and defaults) used by the templates

The nginx job has a fairly straightforward spec:

**jobs/nginx/spec**

    cat <<"EOF" >jobs/nginx/spec
    ---
    name: nginx

    templates:
      nginx_ctl:      bin/nginx_ctl
      nginx.conf.erb: config/nginx.conf
      mime.types:     config/mime.types

    packages:
      - nginx

    properties:
      nginx.workers:
        description: Number of nginx worker processes
        default: 1
      wordpress.servername:
        description: Name of the virtual server
      wordpress.port:
        description: TCP port upstream (backends) servers listen on
        default: 8008
      wordpress.servers:
        description: Array of upstream (backends) servers
    EOF

#### add packages to job spec

The first step toward creating a deployable job is defining the package dependencies in the spec file.  As we determined earlier, the wordpress job requires the apache, wordpress, php, and mysql client packages:

**jobs/wordpress/spec**

    cat <<"EOF" >jobs/wordpress/spec
    ---
    name: wordpress
      
    packages:      
      - mysqlclient
      - apache2
      - php5
      - wordpress      
    EOF

#### create a stub monit config

BOSH uses [monit](http://mmonit.com/monit/) to manage the lifecycle of job processes.  At a bare minimum the monit config file inside the job directory needs to define a pidfile and how to start/stop the job.  It's going to be a lot easier to build out the rest of our job after we've got a vm deployed with  packages installed, so we're going to create a stub monit config:

**jobs/wordpress/monit**
    
    cat <<"EOF" >jobs/wordpress/monit
    check process wordpress
      with pidfile /var/run/stub.pid
      start program "/bin/cp /var/run/monit.pid /var/run/stub.pid" with timeout 60 seconds
      stop program "/bin/rm -f /var/run/stub.pid" with timeout 60 seconds
      group vcap
    EOF

and update our spec:

**jobs/wordpress/spec**

      cat <<"EOF" >jobs/wordpress/spec
      ---
      name: wordpress

      templates: {}

      packages:
        - mysqlclient
        - apache2
        - php5
        - wordpress
      EOF

This will allow us to define a 'wordpress' job in our deployment manifest and deploy a vm with our packages.

#### create templates

The templates directory contains the scripts and config files needed for running the job processes.  BOSH evaluates these files as ruby [ERB templates](http://www.stuartellis.eu/articles/erb/), allowing them to be generalized.  The unique per-deployment information can then be abstracted into properties defined in the deployment manifest.

It generally easier to create templates on an existing BOSH instance that has the required packages already installed.  Start with the configuration files needed by the job packages, and once the processes can be launched manually then move on to writing the control scripts.  When converting existing files/scripts to work on a BOSH managed instance, the most common modifications are usually changing paths to the appropriate location on a BOSH managed system.

**BOSH filesystem conventions**

| object                  | path                             |
| ----------------------- | ---------------------------------|
| package contents        | /var/vcap/package/{package name} |
| job configuration files | /var/vcap/jobs/{job name}/config |
| control scripts         | /var/vcap/jobs/{job name}/bin    |
| pidfiles                | /var/vcap/sys/run                |
| log storage             | /var/vcap/sys/log/{process name} |
| persistent data storage | /var/vcap/store                  |

For example, in the wordpress job the apache configuration needs to set the document root to point at our wordpress package:

    DocumentRoot "/var/vcap/packages/wordpress"

and for the appropriate log file locations:

    ErrorLog "/var/vcap/sys/log/apache2/error_log"
    CustomLog "/var/vcap/sys/log/apache2/access_log" common

Once the configuration file is completed it can be copied into the templates directory and the spec file updated:

**jobs/wordpress/spec**

      cat <<"EOF" >jobs/wordpress/spec
      ---
      name: wordpress

      templates:
        httpd.conf.erb:     config/httpd.conf

      packages:
        - mysqlclient
        - apache2
        - php5
        - wordpress
      EOF

Once all of the necessary configuration files are in place you must write a control script that monit can use to start/stop the process and replace the stub monit config.

TODO: detailed control script example

#### create job properties

At this point you should have a job that can be deployed by BOSH.  The final step is to determine what configuration information should be abstracted into job properties.  Passwords, account names, shared secrets, hostnames, IP addresses and port numbers are all examples of typical job properties.

To reference a job property in a ERB template, use the "p" helper, e.g.:

**jobs/wordpress/templates/wp-config.php.erb**

    cat <<"EOF" >jobs/wordpress/templates/wp-config.php.erb
    define('AUTH_KEY',         '<%= p("wordpress.auth_key") %>');
    define('SECURE_AUTH_KEY',  '<%= p("wordpress.secure_auth_key") %>');
    define('LOGGED_IN_KEY',    '<%= p("wordpress.logged_in_key") %>');
    define('NONCE_KEY',        '<%= p("wordpress.nonce_key") %>');
    define('AUTH_SALT',        '<%= p("wordpress.auth_salt") %>');
    define('SECURE_AUTH_SALT', '<%= p("wordpress.secure_auth_salt") %>');
    define('LOGGED_IN_SALT',   '<%= p("wordpress.logged_in_salt") %>');
    define('NONCE_SALT',       '<%= p("wordpress.nonce_salt") %>');
    EOF

All job properties should be defined in your job spec, along with a description and an optional default value.

**jobs/wordpress/spec**
    
    cat <<"EOF" >>jobs/wordpress/spec
    properties:
      wordpress.auth_key:
        description: Wordpress Authentication Unique Keys (AUTH_KEY)
      wordpress.secure_auth_key:
        description: Wordpress Authentication Unique Keys (SECURE_AUTH_KEY)
      wordpress.logged_in_key:
        description: Wordpress Authentication Unique Keys (LOGGED_IN_KEY)
      wordpress.nonce_key:
        description: Wordpress Authentication Unique Keys (NONCE_KEY)
      wordpress.auth_salt:
        description: Wordpress Authentication Unique Salts (AUTH_SALT)
      wordpress.secure_auth_salt:
        description: Wordpress Authentication Unique Salts (SECURE_AUTH_SALT)
      wordpress.logged_in_salt:
        description: Wordpress Authentication Unique Salts (LOGGED_IN_SALT)
      wordpress.nonce_salt:
        description: Wordpress Authentication Unique Salts (NONCE_SALT)
    EOF

### <a id="deploy"></a>Deploy

To deploy the sample application edit the example deployment manifest (located at the /examples directory) and adapt it with your data. Then use the following command sequence:

    bosh upload release releases/wordpress-2.yml
    bosh deployment <wordpress deployment manifest>
    bosh deploy


#### TODO

* Run through this readme and commit updated code to this repo
* Fill in all missing jobs specs etc. so this README is a complete set of steps to recreate this release from scratch
* When committing update code, recreate git history to match each step in the README?
* Another example bosh-release-simple: Use Redis
* Rename this example bosh-release-intermediate: Rename this release?
* bosh-release-advanced: Use [HyperDB](http://wordpress.org/extend/plugins/hyperdb/installation/) & [MySQL master/slave replication](http://dev.mysql.com/doc/refman/5.1/en/replication-howto.html)
* tmux screencast of this README?
