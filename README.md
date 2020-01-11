# Install Elasticsearch on Raspberry Pi running NextCloudPi

(Note: for editing files I am using emacs-nox as editor, but vi, nano or any other can be used as well, of course.)

## Update the system

```Shell
sudo apt-get update
sudo apt-get upgrade
sudo init 6
```

## Install a JDK

```Shell
sudo apt-get install openjdk-11-jdk
sudo emacs /etc/profile

sudo -s 
cat >> /etc/profile <<EOF
export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:/bin/java::")
export PATH=$JAVA_HOME/bin:$PATH
EOF
exit
```

For installation of Elasticsearch 7.4.2 without JDK the existing one can simply be linked where Elasticsearch otherwise would install it.

```Shell
sudo mkdir -p /usr/share/elasticsearch
sudo ln -s ${JAVA_HOME} /usr/share/elasticsearch/jdk
```

## Install Elasticsearch without JDK

```Shell
cd
mkdir Download
cd Download/

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.4.2-no-jdk-amd64.deb
sudo dpkg -i --force-all --ignore-depends=libc6 elasticsearch-7.4.2-no-jdk-amd64.deb
```

Since the package was installed ignoring the libc6 dependency, the dpkg status is now broken. Future installations will report an error, unless the inconsistency is fixed, e.g. by manually editing the file `/var/lib/dpkg/status`.

Search for the entry `Package: elasticsearch` -> `Depends: [...] libc6, [...]` and remove the dependency `libc6, `.

```Shell
sudo emacs /var/lib/dpkg/status
```

The following alternative to the above did _not_ work for me:

```Shell
# # latest
# wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
# sudo apt-get install apt-transport-https
# echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
# sudo apt-get update && sudo apt-get install elasticsearch
```

Now check the status, logs, fix problems and restart the service

```Shell
sudo systemctl status elasticsearch.service
sudo systemctl restart elasticsearch.service

sudo tail -F /var/log/elasticsearch/elasticsearch.log
sudo journalctl -f --unit elasticsearch
```

## Problems and Fixes

### File Permissions

_Problem_

```Log
Jan 11 16:55:05 nextcloudpi elasticsearch[16825]: Exception in thread "main" org.elasticsearch.bootstrap.BootstrapException: org.elasticsearch.cli.UserException: unable to create temporary keystore at [/etc/elasticsearch/elasticsearch.keystore.tmp], please check filesystem permissions
Jan 11 16:55:05 nextcloudpi elasticsearch[16825]: Likely root cause: java.nio.file.AccessDeniedException: /etc/elasticsearch/elasticsearch.keystore.tmp
```

_Remedy_

```Shell
sudo chgrp elasticsearch /etc/elasticsearch
sudo chmod g+rwx /etc/elasticsearch
```


### JNA Native Support Library

_Problem_

```Log
[2020-01-11T11:26:27,514][WARN ][o.e.b.Natives            ] [nextcloudpi] unable to load JNA native support library, native methods will be disabled.
java.lang.UnsatisfiedLinkError: Native library (com/sun/jna/linux-arm/libjnidispatch.so) not found in resource path (
```

_Remedy_

Create a modified jar-file containing the missing lib and replace the original.

```Shell
mkdir jna
cd jna

cp /usr/share/elasticsearch/lib/jna-4.5.1.jar .
cp jna-4.5.1.jar jna-4.5.1.jar.orig
mkdir jar
cd jar
unzip ../jna-4.5.1.jar.orig

mkdir -p com/sun/jna/linux-arm/
cp /usr/lib/arm-linux-gnueabihf/jni/libjnidispatch.system.so com/sun/jna/linux-arm/libjnidispatch.so
zip -r ../jna-4.5.1.jar *
sudo cp ../jna-4.5.1.jar /usr/share/elasticsearch/lib/jna-4.5.1.jar
```

### X-Pack is not supported

_Problem_

```log
# [2020-01-11T08:39:11,738][ERROR][o.e.b.Bootstrap          ] [nextcloudpi] Exception
# org.elasticsearch.ElasticsearchException: X-Pack is not supported and Machine Learning is not available for [linux-arm]; you can use the other X-Pack features (unsupported) by setting xpack.ml.enabled: false in elasticsearch.yml
```

_Remedy_

Backup the yml file and replace (or add) the related setting.

```shell
sudo cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch_$(date +%Y%m%d%H%M%S).yml
sudo emacs /etc/elasticsearch/elasticsearch.yml
```

```conf
#xpack.ml.enabled: true
xpack.ml.enabled: false
```

### Warning: unable to install syscall filter

Note: this is "just" a warning. You can ignore it, or the syscall protection feature can be disabled completely. Its not working anyway, as you can read. I think this is just not supported by the Raspberry Pi kernel, so we must live without this security feature.

_Problem_

```log
[2020-01-11T12:22:48,433][WARN ][o.e.b.JNANatives         ] [nextcloudpi] unable to install syscall filter: 
java.lang.UnsupportedOperationException: seccomp unavailable: 'arm' architecture unsupported
```

_Remedy_

Backup the yml file and replace (or add) the related setting.

```shell
sudo cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch_$(date +%Y%m%d%H%M%S).yml
sudo emacs /etc/elasticsearch/elasticsearch.yml
```

```conf
#bootstrap.system_call_filter: true
bootstrap.system_call_filter: false
```

### Could not initialize class

_Problem_

```log
[2020-01-11T09:21:15,476][ERROR][o.e.b.ElasticsearchUncaughtExceptionHandler] [nextcloudpi] fatal error in thread [main], exiting
java.lang.NoClassDefFoundError: Could not initialize class com.sun.jna.Native
https://discuss.elastic.co/t/elasticsearch-no-longer-works-under-systemd-7-4-0-on-centos-7-7-1908/201846
```

_Remedy_

Backup the JVM options file and replace (or add) the `tmpdir` setting.

```shell
sudo cp /etc/elasticsearch/jvm.options /etc/elasticsearch/jvm_$(date +%Y%m%d%H%M%S).options
sudo emacs /etc/elasticsearch/jvm.options 
```

```conf
#-Djava.io.tmpdir=${ES_TMPDIR}
-Djava.io.tmpdir=/var/lib/elasticsearch/tmp
```

```shell
sudo mkdir /var/lib/elasticsearch/tmp
sudo chown elasticsearch:elasticsearch /var/lib/elasticsearch/tmp
```

## Links

https://discuss.elastic.co/t/installing-elasticsearch-7-4-on-a-raspberry-pi-4-raspbian-buster/202599
https://medium.com/hepsiburadatech/setup-elasticsearch-7-x-cluster-on-raspberry-pi-asus-tinker-board-6a307851c801
https://www.elastic.co/guide/en/elasticsearch/reference/current/starting-elasticsearch.html
https://discuss.elastic.co/t/x-pack-is-not-supported-and-machine-learning-is-not-available-for-windows-x86/85084
https://stackoverflow.com/a/21151036
https://www.raspberrypi.org/forums/viewtopic.php?t=64803
https://discuss.elastic.co/t/java-lang-unsupportedoperationexception-on-starting-elasticsearch/64548
https://www.elastic.co/guide/en/elasticsearch/reference/master/_system_call_filter_check.html
https://discuss.elastic.co/t/elasticsearch-7-4-wont-start-properly/201891/8