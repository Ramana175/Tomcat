# Apache Tomcat on AWS EC2 — Complete Beginner Guide

A practical, beginner-friendly guide to Java, Tomcat, Maven WAR deployment, Tomcat Manager, EC2 networking, troubleshooting, and hands-on DevOps practice.

> **Two separate examples are covered here:**
>
> - **Main lab — Ubuntu EC2:** Java 21 and Apache Tomcat 10.1.61 installed at `/opt/apache-tomcat-10.1.61`.
> - **Additional lab — Amazon Linux:** Tomcat 9. Use the exact version and archive format currently offered on the official Apache Tomcat website. Do not mix commands or paths between the two operating systems.

## Table of contents

1. [JAR vs WAR vs EAR](#1-jar-vs-war-vs-ear)
2. [What is Tomcat?](#2-what-is-tomcat)
3. [Tomcat vs other application servers](#3-tomcat-vs-other-application-servers)
4. [Prerequisites and EC2 networking](#4-prerequisites-and-ec2-networking)
5. [Install Tomcat 10.1.61 on Ubuntu EC2](#5-install-tomcat-10161-on-ubuntu-ec2)
6. [Tomcat directory structure](#6-tomcat-directory-structure)
7. [Start, stop, and verify Tomcat](#7-start-stop-and-verify-tomcat)
8. [Configure Tomcat Manager and Host Manager](#8-configure-tomcat-manager-and-host-manager)
9. [Change Tomcat port from 8080 to 9090](#9-change-tomcat-port-from-8080-to-9090)
10. [Build and deploy a Maven WAR from another EC2 instance](#10-build-and-deploy-a-maven-war-from-another-ec2-instance)
11. [Troubleshooting](#11-troubleshooting)
12. [Apache HTTP Server vs Tomcat](#12-apache-http-server-vs-tomcat)
13. [Install Tomcat 9 on Amazon Linux](#13-install-tomcat-9-on-amazon-linux)
14. [Security best practices](#14-security-best-practices)
15. [Hands-on assignments](#15-hands-on-assignments)
16. [Official documentation](#16-official-documentation)

---

## 1. JAR vs WAR vs EAR

| Package | Meaning | Typical use | How to run/deploy |
|---|---|---|---|
| JAR | Java Archive | Java library or standalone Java application | An executable JAR can run with `java -jar app.jar` |
| WAR | Web Application Archive | Java web application containing web resources and application classes | Deploy to a compatible servlet container such as Tomcat |
| EAR | Enterprise Archive | Enterprise application that can bundle multiple modules | Deploy to a compatible full Jakarta EE application server, such as WildFly |

Important details:

- **Not every JAR is executable.** `java -jar file.jar` works when the JAR has an entry point, usually the `Main-Class` manifest attribute.
- A **WAR** often contains HTML/JSP pages, web configuration, and compiled Java classes.
- A WAR is not normally run using `java -jar`; it is deployed to a compatible server.
- **Tomcat does not provide full EAR deployment support.** Use a full application server when the application requires enterprise features beyond Tomcat's servlet-container scope.

## 2. What is Tomcat?

Apache Tomcat is an open-source Java web server and servlet container maintained by the Apache Software Foundation. It runs Java web applications that use supported Servlet, Pages, and related web technologies.

Tomcat is commonly used to:

- Host Java web applications packaged as WAR files.
- Receive HTTP requests on a configured port, commonly `8080`.
- Pass requests to the appropriate web application.
- Return the application's response to the client.
- Provide logs to diagnose deployment and runtime problems.

Many Java web applications use the servlet model. Spring Boot servlet-based web applications commonly use embedded Tomcat by default, though the embedded server can be changed.

## 3. Tomcat vs other application servers

| Product | General role | Typical use |
|---|---|---|
| Apache Tomcat | Servlet container / Java web server | Java web applications, commonly WAR files |
| JBoss EAP / WildFly | Full application server | Applications requiring broader Jakarta EE capabilities |
| Oracle WebLogic | Enterprise application server | Enterprise Java applications |
| IBM WebSphere | Enterprise application server family | Enterprise Java workloads |
| Eclipse GlassFish | Jakarta EE application server | Jakarta EE applications |

**Simple rule:** Tomcat is a good fit for many web applications. Choose a full application server when the application needs enterprise APIs or deployment features that Tomcat does not provide. Capabilities vary by version, so check the documentation for the exact version your application requires.

## 4. Prerequisites and EC2 networking

You need an AWS account, an EC2 instance, SSH access, a supported Java version, and a security group that permits only required inbound traffic.

### Example EC2 security group rules

| Type | Protocol | Port | Source |
|---|---|---:|---|
| SSH | TCP | 22 | Your trusted client IP `/32` |
| Custom TCP | TCP | 8080 | Your trusted client IP `/32` |

If you later change Tomcat to port `9090`, allow TCP `9090` from your trusted client IP and remove the old `8080` rule if no longer required.

Avoid exposing Tomcat Manager or Host Manager to the entire internet. Use a trusted IP range or private access method.

Find your EC2 public IPv4 address in the AWS EC2 console under **Instances**. Public addresses can change after stopping and starting an instance unless you use an Elastic IP.

Replace placeholders such as `YOUR_EC2_PUBLIC_IP` with the current address of your instance.

## 5. Install Tomcat 10.1.61 on Ubuntu EC2

This section describes the main lab.

### 5.1 Check Java

```bash
java -version
javac -version
```

The lab environment used Java 21. Check the Tomcat 10.1 documentation for the requirements of your selected release.

If Java is missing, install a suitable JDK on Ubuntu:

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
```

### 5.2 Download and extract Tomcat

Official Tomcat 10 downloads: <https://tomcat.apache.org/download-10.cgi>

Obtain the matching `10.1.61` archive from the official Apache site and verify its checksum or signature where available. If the archive is in your home directory:

```bash
cd ~
ls -lh apache-tomcat-10.1.61.tar.gz
```

Extract it:

```bash
sudo tar -xzf apache-tomcat-10.1.61.tar.gz -C /opt
ls -ld /opt/apache-tomcat-10.1.61
```

If your downloaded archive has a different filename, use that exact filename. Do not combine a URL for one version with a filename for another.

### 5.3 Installation path

The main lab's Tomcat home is:

```text
/opt/apache-tomcat-10.1.61
```

Important directories include `bin`, `conf`, `logs`, `webapps`, `lib`, `temp`, and `work`.

## 6. Tomcat directory structure

```text
/opt/apache-tomcat-10.1.61/
├── bin/       # Startup, shutdown, and administration scripts
├── conf/      # server.xml, web.xml, users, and other configuration
├── lib/       # Shared Java libraries
├── logs/      # Catalina and other logs
├── temp/      # Temporary files
├── webapps/   # Applications deployed to Tomcat
└── work/      # Generated work files, including compiled JSP artifacts
```

| Path | Purpose |
|---|---|
| `bin/startup.sh` | Starts Tomcat |
| `bin/shutdown.sh` | Requests Tomcat to stop |
| `bin/catalina.sh` | Main Tomcat control script |
| `conf/server.xml` | Connectors, ports, and server configuration |
| `conf/tomcat-users.xml` | Local users and roles for supported Tomcat applications |
| `webapps/` | Default application deployment directory |
| `webapps/ROOT/` | Root context, normally opened at `/` |
| `webapps/manager/` | Tomcat Web Application Manager, if installed |
| `webapps/host-manager/` | Virtual-host administration app, if installed |
| `logs/catalina.out` | Common output log for shell-script startup on Unix-like systems |

Installed files can vary depending on the distribution and installation method.

## 7. Start, stop, and verify Tomcat

Use the full path to avoid confusion about your current directory.

### Start

```bash
sudo /opt/apache-tomcat-10.1.61/bin/startup.sh
```

### Stop

```bash
sudo /opt/apache-tomcat-10.1.61/bin/shutdown.sh
```

### Check the listener

```bash
sudo ss -lntp | grep ':8080'
```

### Test locally on EC2

```bash
curl -I http://localhost:8080/
```

A successful response commonly starts with an HTTP status such as `HTTP/1.1 200`.

### Open in a browser

```text
http://YOUR_EC2_PUBLIC_IP:8080/
```

This works only if Tomcat is running, the instance is reachable, and security/network rules permit the connection.

### Check the version and logs

```bash
sudo /opt/apache-tomcat-10.1.61/bin/version.sh
sudo tail -n 100 /opt/apache-tomcat-10.1.61/logs/catalina.out
```

## 8. Configure Tomcat Manager and Host Manager

**Security warning:** Manager and Host Manager provide administrative access. Use strong unique passwords, restrict access by source IP, and never commit credentials to Git.

### 8.1 Add a Manager user

Edit the existing users file:

```bash
sudo nano /opt/apache-tomcat-10.1.61/conf/tomcat-users.xml
```

Inside the existing `<tomcat-users>` root element, add the role and user before its closing tag:

```xml
<role rolename="manager-gui"/>
<user username="tomcatadmin" password="REPLACE_WITH_A_STRONG_UNIQUE_PASSWORD" roles="manager-gui"/>
```

Replace the example password with a long, unique password. Do not add a second root element or place these entries outside the existing root element.

### 8.2 Add a Host Manager user if needed

```xml
<role rolename="admin-gui"/>
<user username="hostadmin" password="REPLACE_WITH_ANOTHER_STRONG_UNIQUE_PASSWORD" roles="admin-gui"/>
```

Add these lines inside the same existing `<tomcat-users>` element. Use a separate strong password. Host Manager is not needed for a basic WAR deployment.

### 8.3 Restrict Manager access by client IP

Inspect the Manager context configuration:

```bash
sudo nano /opt/apache-tomcat-10.1.61/webapps/manager/META-INF/context.xml
```

Tomcat commonly restricts remote access by default using a `RemoteAddrValve`. Preserve the restriction and add your trusted client public IP to the existing allow expression rather than commenting out the valve.

For example, an illustrative pattern for a client IP of `203.0.113.25` is:

```xml
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|203\.0\.113\.25"
```

`203.0.113.25` is a documentation example, **not your actual IP**. Replace it with your current client public IP. Literal dots in an IPv4 regular-expression pattern should be escaped as `\.`. Check the exact syntax in your installed file before editing.

If your public client IP changes, update the allow expression. Do not allow every address merely to make the page load.

### 8.4 Restrict Host Manager access by client IP

If you use Host Manager, inspect:

```bash
sudo nano /opt/apache-tomcat-10.1.61/webapps/host-manager/META-INF/context.xml
```

Apply a similar trusted-IP restriction to its existing valve, preserving the file structure. Do not replace the entire configuration without checking it first.

### 8.5 Restart and open the applications

```bash
sudo /opt/apache-tomcat-10.1.61/bin/shutdown.sh
sudo /opt/apache-tomcat-10.1.61/bin/startup.sh
```

Manager:

```text
http://YOUR_EC2_PUBLIC_IP:8080/manager/html
```

Host Manager:

```text
http://YOUR_EC2_PUBLIC_IP:8080/host-manager/html
```

Access requires both the AWS/network rules and Tomcat's own IP restriction to permit the request.

- HTTP `403`: check the client IP allowlist and logs.
- HTTP `401`: check credentials and roles.

Do not disable access restrictions as a troubleshooting shortcut.

## 9. Change Tomcat port from 8080 to 9090

This can help when another service, such as Jenkins, already occupies port `8080`.

### 9.1 Check whether port 9090 is available

```bash
sudo ss -lntp | grep ':9090'
```

No output usually means no process is currently listening on that port. Confirm it is suitable for your environment.

### 9.2 Back up and edit `server.xml`

```bash
sudo cp /opt/apache-tomcat-10.1.61/conf/server.xml /opt/apache-tomcat-10.1.61/conf/server.xml.bak
sudo nano /opt/apache-tomcat-10.1.61/conf/server.xml
```

Find the existing HTTP connector containing `port="8080"`, for example:

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

Change the existing connector port to `9090`:

```xml
<Connector port="9090" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

The exact formatting and attributes may differ. Edit the existing HTTP connector; do not create a duplicate.

### 9.3 Restart and test

```bash
sudo /opt/apache-tomcat-10.1.61/bin/shutdown.sh
sudo /opt/apache-tomcat-10.1.61/bin/startup.sh
sudo ss -lntp | grep ':9090'
curl -I http://localhost:9090/
```

Allow TCP `9090` from your trusted client IP in the EC2 security group, then open:

```text
http://YOUR_EC2_PUBLIC_IP:9090/
```

Use port `9090` in application and Manager URLs too. If Tomcat fails to start, inspect the logs and restore the backup if necessary.

## 10. Build and deploy a Maven WAR from another EC2 instance

This workflow uses one EC2 instance to build the WAR and another EC2 instance to run Tomcat.

```text
Developer / Git repository
          |
          v
Maven build EC2
  mvn clean package
          |
          v
      WAR artifact
          |
          v
Tomcat EC2
  /opt/apache-tomcat-10.1.61/webapps/
          |
          v
Browser / HTTP request
```

Example paths from the lab:

- Maven project: `/home/ubuntu/Maven/tomcat-web-application`
- WAR filename: `tomcat-web-application-1.0-SNAPSHOT.war`
- Tomcat home: `/opt/apache-tomcat-10.1.61`

Adjust paths and filenames to match your actual environment.

### 10.1 Build on the Maven EC2 instance

```bash
cd /home/ubuntu/Maven/tomcat-web-application
mvn clean package
ls -lh target/*.war
```

Resolve build errors before transferring the artifact.

### 10.2 Transfer the WAR to Tomcat EC2

From the build instance, use `scp` with a valid private-key path:

```bash
scp -i /path/to/your-key.pem target/tomcat-web-application-1.0-SNAPSHOT.war ubuntu@YOUR_TOMCAT_EC2_PUBLIC_IP:/home/ubuntu/
```

Replace the key path and public IP. Keep the private key secure. You can alternatively use an approved artifact repository or transfer method.

### 10.3 Copy the WAR into Tomcat's `webapps` directory

SSH to the Tomcat EC2 instance:

```bash
sudo cp /home/ubuntu/tomcat-web-application-1.0-SNAPSHOT.war /opt/apache-tomcat-10.1.61/webapps/tomcat-web-application.war
sudo ls -lh /opt/apache-tomcat-10.1.61/webapps/tomcat-web-application.war
```

The destination filename normally creates the context path `/tomcat-web-application`. Tomcat commonly deploys WAR files automatically when deployment is enabled. If it does not deploy, inspect logs first. During a learning lab, you can restart Tomcat:

```bash
sudo /opt/apache-tomcat-10.1.61/bin/shutdown.sh
sudo /opt/apache-tomcat-10.1.61/bin/startup.sh
```

### 10.4 Open the application

```text
http://YOUR_TOMCAT_EC2_PUBLIC_IP:8080/tomcat-web-application/
```

If you changed the port to `9090`, use that port. Ensure the security group permits the selected HTTP port from your client. Keep SSH restricted.

### 10.5 Example `index.html`

A basic page can be stored at `src/main/webapp/index.html` in a conventional Maven WAR project:

```html
<!DOCTYPE html>
<html>
<head>
  <title>My Tomcat Application</title>
</head>
<body>
  <h1>Welcome to My Tomcat Web Application!</h1>
  <p>My Java web application is running successfully on Apache Tomcat.</p>
  <p><a href="hello">Click here to run my Java Servlet</a></p>
</body>
</html>
```

The `hello` link works only if the application includes a servlet mapped to `/hello`. Static HTML alone does not create that servlet.

## 11. Troubleshooting

### Browser times out

Check in this order:

1. Is Tomcat running?
2. Is it listening on the expected port?
3. Does `curl` work from inside EC2?
4. Does the security group allow the port from your client IP?
5. Are network ACLs, routes, host firewalls, or other controls blocking the request?
6. Are you using the current public IP?

```bash
sudo ss -lntp | grep -E ':8080|:9090'
curl -I http://localhost:8080/
```

Use the correct port in `curl`.

### `Address already in use` or `BindException`

Another process may already use the port:

```bash
sudo ss -lntp | grep ':8080'
```

Stop or reconfigure the conflicting service only if appropriate, or change Tomcat's HTTP connector to an unused port using Section 9.

### `Permission denied` while inspecting a deployed file

Use `sudo` for diagnosis when the file is owned by another user:

```bash
sudo ls -l /opt/apache-tomcat-10.1.61/webapps/tomcat-web-application/
sudo ls -lh /opt/apache-tomcat-10.1.61/webapps/tomcat-web-application.war
```

Do not use `chmod 777` as a shortcut. Fix the specific ownership or permission only after understanding which account runs Tomcat and what access is needed.

### WAR is not deployed

```bash
sudo ls -lh /opt/apache-tomcat-10.1.61/webapps/
sudo tail -n 100 /opt/apache-tomcat-10.1.61/logs/catalina.out
sudo jar tf /opt/apache-tomcat-10.1.61/webapps/tomcat-web-application.war | head -n 40
```

Confirm the WAR is valid, expected web resources are present, and the application is compatible with your Tomcat version. Review deployment errors in the logs.

### Manager returns HTTP 403

The Manager application's IP restriction may be rejecting your client IP. Inspect `webapps/manager/META-INF/context.xml`, check the IP seen by the server, update the allowlist carefully, and keep the AWS security group restricted too.

### Manager returns HTTP 401

Check the username, password, and role in `conf/tomcat-users.xml`. Ensure the role is inside the existing `<tomcat-users>` element. Restart Tomcat after configuration changes if needed.

### `curl localhost` works but the browser does not

Tomcat may be healthy while the network path is blocked. Check the public IP, security group, port, routing, and any host firewall. `localhost` tests only the instance itself.

### Tomcat fails after a configuration edit

```bash
sudo tail -n 100 /opt/apache-tomcat-10.1.61/logs/catalina.out
```

Check `server.xml` for malformed XML, duplicate connector ports, or invalid configuration. Restore the backup only if it is the correct known-good copy:

```bash
sudo cp /opt/apache-tomcat-10.1.61/conf/server.xml.bak /opt/apache-tomcat-10.1.61/conf/server.xml
```

## 12. Apache HTTP Server vs Tomcat

| Feature | Apache HTTP Server (`httpd` / `apache2`) | Apache Tomcat |
|---|---|---|
| Main role | General-purpose web server | Java servlet container and web server |
| Common content | Static files, reverse proxying, TLS termination | Java web applications and servlet-based dynamic content |
| Typical ports | `80` for HTTP, `443` for HTTPS | `8080` by default in many configurations |
| Common content/application directory | `/var/www/html` on common Linux packages | `webapps/` in a standard Tomcat installation |
| Ubuntu package/service | `apache2` | Often installed from an archive or package |
| Amazon Linux package/service | `httpd` | Often installed from an archive or package |

These roles can overlap. A common production design uses Apache HTTP Server or Nginx in front of Tomcat as a reverse proxy, with TLS and routing handled at the front end.

### Install Apache HTTP Server on Amazon Linux

```bash
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

Allow inbound TCP `80` in the EC2 security group from the intended audience. The default document root is commonly `/var/www/html`.

### Install on Ubuntu

On Ubuntu, the package and service are normally called `apache2`:

```bash
sudo apt update
sudo apt install -y apache2
sudo systemctl enable --now apache2
sudo systemctl status apache2
```

## 13. Install Tomcat 9 on Amazon Linux (additional lab)

This is a separate example. Do not use its package commands or paths for the Ubuntu Tomcat 10.1.61 lab.

### 13.1 Install prerequisites and Java

On an Amazon Linux version that supports `yum`:

```bash
sudo yum update -y
sudo yum install -y wget unzip tar tree
```

Install a Java version supported by the selected Tomcat 9 release and your application. Package names differ by Amazon Linux release. For example, if OpenJDK 21 is available in your repository:

```bash
sudo yum install -y java-21-amazon-corretto-devel
java -version
```

If the package is unavailable, consult the AWS documentation for your specific release.

### 13.2 Download the matching Tomcat archive

Official Tomcat 9 downloads: <https://tomcat.apache.org/download-90.cgi>

Choose a release and archive format from the official page. Make sure the URL, version, and filename match. For a `.tar.gz` archive, the workflow is:

```bash
cd /tmp
# Download the exact archive URL shown on the official Apache Tomcat page.
# Replace VERSION with the exact release you selected.
sudo tar -xzf apache-tomcat-VERSION.tar.gz -C /opt
```

The extracted directory normally matches the archive version, such as `/opt/apache-tomcat-9.0.x`. Replace `9.0.x` below with the actual installed directory name.

### 13.3 Start and verify

```bash
sudo /opt/apache-tomcat-9.0.x/bin/startup.sh
sudo ss -lntp | grep ':8080'
curl -I http://localhost:8080/
```

To stop:

```bash
sudo /opt/apache-tomcat-9.0.x/bin/shutdown.sh
```

Allow inbound port `8080` only from the required source range. Tomcat 9 and Tomcat 10 use different Servlet/Jakarta namespace generations, so check application compatibility before deploying.

## 14. Security best practices

- Restrict SSH (TCP `22`) to your trusted client IP or approved access path.
- Restrict Tomcat's HTTP port to trusted sources while learning.
- Never expose Manager or Host Manager to `0.0.0.0/0`.
- Use strong, unique administrative passwords.
- Never commit passwords, private keys, or other secrets to GitHub.
- Follow least privilege for operating-system accounts and Tomcat roles.
- Do not use `chmod 777` to fix deployment problems.
- Keep Java, Tomcat, the operating system, and dependencies patched.
- Download Tomcat from the official Apache site and verify signatures/checksums when available.
- Remove example and unused administrative applications from production installations when appropriate.
- Use HTTPS for real deployments; do not send administrative credentials over untrusted plain HTTP.
- Back up configuration files before changing them.
- Review logs after each deployment or configuration change.
- For production, consider a reverse proxy, centralized logging, monitoring, backups, and a rollback-capable deployment process.

## 15. Hands-on assignments

Complete one task at a time and verify the result before moving to the next.

### Assignment 1 — Basic Tomcat

- [ ] Check `java -version`.
- [ ] Start Tomcat.
- [ ] Confirm the listening port with `ss`.
- [ ] Test with `curl -I http://localhost:8080/`.
- [ ] Open the page from your browser.

### Assignment 2 — Explore directories and logs

- [ ] List `bin`, `conf`, `logs`, and `webapps`.
- [ ] Find `server.xml`.
- [ ] Inspect `catalina.out`.
- [ ] Explain each directory in your own words.

### Assignment 3 — Secure Manager access

- [ ] Create a user with only the required Manager role.
- [ ] Restrict Manager access to your current trusted client IP.
- [ ] Restrict the EC2 security group to the same trusted source where possible.
- [ ] Confirm authorized access works.
- [ ] If safe, verify access from an unapproved source is blocked.

### Assignment 4 — Change the port

- [ ] Back up `server.xml`.
- [ ] Change the HTTP connector to `9090`.
- [ ] Restart Tomcat.
- [ ] Confirm the listener with `ss`.
- [ ] Update the security group and test the new URL.
- [ ] Explain why two services normally cannot bind to the same IP and TCP port.

### Assignment 5 — Maven WAR deployment

- [ ] Run `mvn clean package`.
- [ ] Confirm the WAR exists under `target/`.
- [ ] Transfer it to the Tomcat EC2 instance.
- [ ] Copy it into Tomcat's `webapps/` directory.
- [ ] Check deployment logs.
- [ ] Open the application URL.

### Assignment 6 — Troubleshooting

- [ ] Diagnose a wrong port.
- [ ] Diagnose an HTTP 403 from Manager.
- [ ] Diagnose a missing WAR or wrong context path.
- [ ] Use logs to identify a deployment error.
- [ ] Fix permissions without using `chmod 777`.

### Assignment 7 — Compare web server and application server

- [ ] Install Apache HTTP Server on a test instance.
- [ ] Serve a static HTML page.
- [ ] Compare its document root with Tomcat's `webapps/`.
- [ ] Explain when a reverse proxy may be useful.

## 16. Official documentation

- Apache Tomcat 10.1: <https://tomcat.apache.org/tomcat-10.1-doc/>
- Apache Tomcat 9: <https://tomcat.apache.org/tomcat-9.0-doc/>
- Tomcat 10 downloads: <https://tomcat.apache.org/download-10.cgi>
- Tomcat 9 downloads: <https://tomcat.apache.org/download-90.cgi>
- Tomcat Manager: <https://tomcat.apache.org/tomcat-10.1-doc/manager-howto.html>
- Tomcat Host Manager: <https://tomcat.apache.org/tomcat-10.1-doc/host-manager-howto.html>
- Tomcat security considerations: <https://tomcat.apache.org/tomcat-10.1-doc/security-howto.html>
- AWS EC2 security groups: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html>
- AWS connect to Linux instances: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html>
- Apache HTTP Server documentation: <https://httpd.apache.org/docs/>
- Apache Maven guides: <https://maven.apache.org/guides/>

## Project status checklist

Use this to track your actual environment. Mark a step complete only after verifying it.

- [ ] Java installed and verified.
- [ ] Tomcat starts and listens on the intended port.
- [ ] EC2 security group allows only required traffic.
- [ ] Manager access is restricted and credentials are secure.
- [ ] Maven WAR builds successfully.
- [ ] WAR deploys and opens at the expected context path.
- [ ] Logs are checked after deployment.
- [ ] Port-change test completed, if required.
- [ ] Troubleshooting assignments completed.

**Remember:** Use the actual EC2 public IP, client IP, archive filename, and installation directory from your environment. Public IPs and software versions can change; verify them before using commands.
