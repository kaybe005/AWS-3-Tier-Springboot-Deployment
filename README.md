# Hands-On AWS 3-Tier Web Application Deployment

A hands-on DevOps project demonstrating the manual deployment and configuration of a **3-tier Spring Boot web application** onto AWS cloud infrastructure using **Nginx**, **Apache Tomcat 9**, and **AWS RDS (MySQL)**.

---

## 🏗️ Architecture Overview

```text
               +-------------------------------------------------------+
               |                  AWS Public Subnet                    |
               |                                                       |
               |   +-------------------+       +-------------------+   |
[Client] ----->|   | Nginx Web Server  | ----> | Tomcat App Server |   |
(Port 80)      |   |   (Port 80)       | Local |   (Port 8080)     |   |
               |   +-------------------+       +-------------------+   |
               +-----------------------------------------|-------------+
                                                         |
                                                  Port 3306 (MySQL)
                                                         |
                                                         v
               +-------------------------------------------------------+
               |                  AWS Private Subnet                   |
               |                                                       |
               |             +---------------------------+             |
               |             |   AWS RDS MySQL Database  |             |
               |             |                           |             |
               |             +---------------------------+             |
               +-------------------------------------------------------+

```

Tech Stack

    Cloud Infrastructure: AWS EC2 (Ubuntu 22.04 LTS), AWS RDS (MySQL 8.0)

    Web & Application Servers: Nginx, Apache Tomcat 9.0.83

    Build & Runtimes: OpenJDK 11, Apache Maven 3.x, Java Spring Boot

    Local Development Environment: macOS, Terminal CLI, Git

🛠️ Hands-On Implementation Steps

1. Database Tier Provisioning (AWS RDS)

   Provisioned an AWS RDS MySQL instance and configured Security Group inbound rules to restrict port 3306 traffic exclusively to the EC2 application instance.

   Initialized the relational database (javaapp) and verified core schemas and permissions using the MySQL CLI client.

2. Local Application Build & Maven Configuration

   Corrected Maven dependencies inside pom.xml by explicitly declaring coordinates for the MySQL connector (mysql-connector-j).

   Handled embedded container exclusions to prevent runtime ClassLoader conflicts during web application context initialization.

   Packaged the application into a standalone web archive:

```
mvn clean package -DskipTests
```

Transferred the target artifact (dptweb-1.0.war) securely from local macOS disk to the EC2 server using scp:
scp -i /path/to/mykey.pem target/dptweb-1.0.war ubuntu@<EC2_PUBLIC_IP>:/home/ubuntu/

3. Application Tier Configuration (Tomcat 9)

   Provisioned Apache Tomcat 9 under /opt/tomcat and configured executable permissions across control scripts.

   Created a custom systemd service file (/etc/systemd/system/tomcat.service) pointing to OpenJDK 11 binaries with custom heap memory constraints (-Xms512M -Xmx1024M).

   Deployed the application binary as ROOT.war inside /opt/tomcat/webapps/ to serve the application at the root context path (/).

4. Reverse Proxy Setup (Nginx)

   Installed Nginx to act as a front-facing HTTP reverse proxy accepting external traffic on port 80.

   Configured upstream proxy pass routing to forward incoming requests internally to Tomcat on 127.0.0.1:8080.

   Applied $http_host proxy header pass-throughs to satisfy HTTP specification constraints and prevent Host header parsing errors in Tomcat.

🔍 Real-World Troubleshooting & Lessons Learned
Issue 1: Systemd Service Exit Code (status=217/USER)

    Symptom: tomcat.service failed to launch during systemd startup with Process /opt/tomcat/bin/startup.sh could not be executed.

    Root Cause: Systemd service definition specified User=tomcat and Group=tomcat, but the dedicated system service account did not exist on the Ubuntu instance.

    Resolution: Updated service unit directives to run under the existing system user (User=ubuntu, Group=ubuntu) and reset directory ownership across /opt/tomcat.

Issue 2: Servlet Container Fragment Collision (org_apache_jasper_el)

    Symptom: Tomcat returned an HTTP 404 Not Found error upon context initialization. catalina.out threw java.lang.IllegalArgumentException: More than one fragment with the name [org_apache_jasper_el] was found.

    Root Cause: Duplicate Expression Language (EL) libraries present in WEB-INF/lib/. Spring Boot pulled tomcat-embed-el, while explicit tomcat-jasper dependencies pulled tomcat-jasper-el, causing runtime Servlet specification violations.

    Resolution: Added an <exclusions> rule in pom.xml under tomcat-jasper to suppress embedded EL duplication before re-packaging the WAR artifact:

    <dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-jasper</artifactId>
    <version>9.0.31</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.tomcat.embed</groupId>
            <artifactId>tomcat-embed-el</artifactId>
        </exclusion>
    </exclusions>
    </dependency>

    Issue 3: Nginx Proxy Header Invalid Host Parsing

    Symptom: Nginx logged 502 Bad Gateway / Tomcat logged The host [_] is not valid.

    Root Cause: Nginx passed server_name _; directly to Tomcat inside the Host header, which Tomcat 9 strict HTTP validation rules explicitly reject.

    Resolution: Replaced proxy_set_header Host $host; with proxy_set_header Host $http_host; inside /etc/nginx/conf.d/app.conf to preserve valid client incoming Host metadata.

    ⚙️ Nginx Reverse Proxy Configuration (/etc/nginx/conf.d/app.conf)

Nginx

upstream backend {
server 127.0.0.1:8080;
}

server {
listen 80;
server*name *;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

}

📌 Verification & Testing

    Nginx Routing: HTTP GET requests to http://<EC2_PUBLIC_IP> return HTTP status 200 OK without requiring port specifiers.

    Spring MVC Mapping: Confirmed DispatcherServlet initialization and routing mapping inside /opt/tomcat/logs/catalina.out:
    Plaintext

    o.s.web.servlet.DispatcherServlet : Initializing Servlet 'dispatcherServlet'
    o.s.web.servlet.DispatcherServlet : Completed initialization in 2 ms

    Database Connectivity: Successfully submitted web form queries from the UI to perform read/write transactions against the AWS RDS MySQL backend.

Note: Application source code adapted for hands-on Linux, Web Server, and AWS infrastructure deployment exercises.
