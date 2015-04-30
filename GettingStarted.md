# Table of Contents #



# Introduction #

**jmxeval** is a plugin for Nagios/NRPE which can query JMX attributes or return values of JMX operations

# Setting Up #

## With Nagios/NRPE ##

To setup **jmxeval** with your Nagios/NRPE setup,
  1. Download the latest version's zip file
  1. Unpack the zip file
  1. Place the following files from the zip file to `libexec` directory under Nagios directory, or any other location you prefer to place Nagios plugins
    * `jmxeval-<version>.jar`
    * `check_jmxeval` (for linux or any unix based platforms)
    * `check_jmxeval.bat` (for windows)
  1. Ensure `JAVA_HOME` environment property is set to a valid Java installation, or change the `check_jmxeval` / `check_jmxeval.bat` to set the `JAVA_HOME` value
  1. Run the following command to make sure setup is complete

**On Linux (or other unix based platforms)**
```
<nagios-dir>/libexec/check_jmxeval
```

**On Windows**
```
<nagios-dir>/libexec/check_jmxeval.bat
```

This should print the following on the screen.

```
Syntax: check_jmxeval <filename> [--validate=true [--schema=x.x]]
```

## As a standalone tool ##

To setup **jmxeval** with your Nagios/NRPE setup,
  1. Download the latest version's zip file
  1. Unpack the zip file
  1. Ensure `JAVA_HOME` environment property is set to a valid Java installation, or change the `check_jmxeval` / `check_jmxeval.bat` to set the `JAVA_HOME` value
  1. Run the following command to make sure setup is complete

**On Linux (or other unix based platforms)**
```
<nagios-dir>/libexec/check_jmxeval
```

**On Windows**
```
<nagios-dir>/libexec/check_jmxeval.bat
```

This should print the following on the screen.

```
Syntax: check_jmxeval <filename> [--validate=true [--schema=x.x]]
```

# Configuring JMX #

To query with **jmxeval**, you will have to ensure that the Java process that you intend to query has JMX enabled. If its already configured, you can skip this section of the document, otherwise you can setup JMX on any Java processes (Java 1.5+) as stated in the following link.

http://docs.oracle.com/javase/1.5.0/docs/guide/management/agent.html#jmxagent

Make sure that you refer the correct version of documentation to determine how to enable JMX, as they could be different, specially if you are still using Java 1.4 which does not support JMX out-of-the-box.

Additionally, you can refer documentation related your tomcat, spring framework, etc. to determine other ways enabling JMX and exposing more useful information related to the application.

# Your first jmxeval check configuration #

Before getting in to details about all the checks that **jmxeval** can perform, lets start with a simple check which checks for the current thread count in a Java process.

This check will include,
  1. Initiating a JMX connection to a JMX enabled Java process, by authenticating itself with a given username and a password (without SSL)
  1. Querying the `ThreadCount` on the `java.lang:type=Threading` MBean
  1. Checking if the thread count is in the expected range
  1. Including the thread count in performance data which Nagios could use to produce history graphs

To perform this check, first you will need to make sure that you have configuration file that instructs **jmxeval** to perform this check. Following is a sample configuration file you could use for this.

```
<?xml version="1.0"?>
<jmxeval:jmxeval xmlns:jmxeval="http://www.adahas.com/schema/jmxeval-1.2" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <!--
    connect to Java process running on 'localhost' having JMX agent exposed on port '9010'
    using username 'myUser' and password 'myPassword'
   -->
  <connection url="service:jmx:rmi:///jndi/rmi://localhost:9010/jmxrmi" 
      username="myUser" password="myPassword">
      
    <!-- 
      define a JMX check to evaluate the thread count of the process named as 'Threads'
     -->
    <eval name="Threads">
    
      <!-- 
        query the JMX connection to get the attribute 'ThreadCount' from the JMX bean with object name
        'java.lang:type=Threading' and assign the retrieved value to a variable named 'threadCount'
       -->
      <query var="threadCount" objectName="java.lang:type=Threading" attribute="ThreadCount" />
      
      <!--
        perform a check using the 'threadCount' variable to see if the value of the variable is
        in the expected range.
        
        if the value is equals to or is over 30 the check will result in a warning status, where as if the
        value is equals to or is over 40 the check will result in critical status.
        
        the message displayed with the plugin output would be 'ThreadCount is ${threadCount}' where the
        '${threadCount}' would be replaced with the value of the variable.
       -->
      <check useVar="threadCount" warning="30" critical="40" message="ThreadCount is ${threadCount}">
      
        <!-- 
          the variable used in this check, in this instance 'threadCount', will be included in the
          performance data produced from the plugin output.  
         -->
        <perf />
      </check>
    </eval>
  </connection>
</jmxeval:jmxeval>
```

To get started, copy the content from the above sample, and place it in an `<nagios-dir>/etc/jmxeval-threadcount.xml` or any other location you prefer.

Then make sure that you have a JMX enabled Java process is running on your computer (or any other computer that is accessible from yours), and the XML file is updated with the proper `url`, `username` and `password` for the `connection`. Also make sure that the `java.lang:type=Threading` MBean is exposed (usually its exposed on all Java processes) using JConsole or VisualVM, and if its not available, update the `objectName` and `attribute` to reflect another MBean that is available.

Now to test if the **jmxeval** check works, execute the following command.

```
<nagios-dir>/libexec/check_jmxeval <nagios-dir>/etc/jmxeval-threadcount.xml
```

And it was successful, it should give a output similar to,

```
JMXEval Threads OK - ThreadCount is 23 | threadCount=23;30;40 time=0.0s
```

# Configuration explained #

The **jmxeval** configuration is supplied as an XML file, which let you configure a simple JMX attribute check to a more advanced checks using multiple JMX attributes/return values from JMX operations, regular expression based status checks and mathematical computations.

The XML configuration file is structured as follows.

```
<?xml version="1.0"?>
<jmxeval:jmxeval xmlns:jmxeval="http://www.adahas.com/schema/jmxeval-1.2"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <!--
    <connection> to use for evaluations defined under this connection
    (multiple connection elements can be used to query multiple processes)
   -->
  <connection ... >
      
    <!-- 
      defines a <eval>uation that will be carried out, which will reported in
      the plugin output
      
      when multiple <eval> elements are defined, the plugin will treat them as
      a multiple checks, and will return the output as a multi-line check
     -->
    <eval ...>
    
      <!-- 
        within an <eval< element, a combination of <query>, <exec> or <expr> elements can be used
        and multiples of these elements are allowed as well
        
        the last element under <eval> element should be a check element, and one <check> element
        is allowed per <eval> element
       -->
       
      <!-- 
        <query> element queries and retrieves a value from a JMX connection, which could be a
        simple attribute of a MBean or an attribute of a composite attribute, and defines a variable
        with the attribute value
       -->
      <query ... >

        <!-- 
          <perf> element flags that the variable created in enclosing <query> element should be
          included in the performance data section of the plugin output
         -->
        <perf ... />
      
      </query>
       
      <!-- 
        <exec> element invokes a operation exposed on JMX and captures the return value, and defines
        a variable with the value captured
       -->
      <exec ... >

        <!-- 
          <perf> element flags that the variable created in enclosing <exec> element should be
          included in the performance data section of the plugin output
         -->
        <perf ... />
      
      </exec>
       
      <!-- 
        <expr> element invokes a pre-defined mathemtical expression using defined values or variables
        defined as a result of <query>, <exec> or other <expr> elements, and defines a new variable
        with the resulting value
       -->
      <expr ... >

        <!-- 
          <perf> element flags that the variable created in enclosing <expr> element should be
          included in the performance data section of the plugin output
         -->
        <perf ... />
      
      </expr>
      
      <!-- 
        <check> element is the mandatory last element on a <eval> element
        
        this performs a comparison check using a variable defined using any of the <query>, <exec> or <expr>
        elements based on given warning and critical levels/patterns and determines the output status
        of the container <eval>ualtion
       -->
      <check ...>
      
        <!-- 
          <perf> element flags that the variable used in enclosing <check> element should be
          included in the performance data section of the plugin output
         -->
        <perf ... />
        
      </check>
      
    </eval>
    
  </connection>

</jmxeval:jmxeval>
```

Above is a sample XML which shows all the placements of different elements. If you are familiar with XML Schema definitions, you can refer to XSD at,

http://code.google.com/p/jmxeval/source/browse/trunk/src/main/resources/schema/jmxeval-1.2.xsd

The XML configuration file is structured to give maximum possible configurability for the checks, most of the elements are not essential for standard usage. For example, a check for one JMX attribute can be written with just a few lines as follows.

```
<?xml version="1.0"?>
<jmxeval:jmxeval xmlns:jmxeval="http://www.adahas.com/schema/jmxeval-1.2"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <connection ... >      
    <eval ...>
      <query ... />
      <check ... />
    </eval>
  </connection>
</jmxeval:jmxeval>
```

## Configuration elements ##

Let us look at each element and what attributes you can configure to implement your **jmxeval** check.

### `<connection>` element ###

The `<connection>` element will initiate a JMX connection for querying for MBean attributes of executing JMX operations configured within the `<connection>` element. It supports JMX authentication using username and password, as secured JMX connections over SSL.

Configuration attributes available on `<connection>` element.
  * `url` - JMX connection URL. This is a mandatory attribute.
  * `username` - Username for authentication.
  * `password` - Password for the given username.
  * `ssl` - Whether to use SSL for the connection.

Here's a few example connection elements.

**Connection to a Java process running on localhost on port 9010, without authentication.
```
<connection url="service:jmx:rmi:///jndi/rmi://localhost:9010/jmxrmi">
```**

**Connection to a Java process running on localhost on port 9010, with authentication.
```
<connection url="service:jmx:rmi:///jndi/rmi://localhost:9010/jmxrmi"
    username="myUser" password="myPassword">
```**

**Connection to a Java process running on host named webserver1.adahas.com on port 9010, with authentication and SSL.
```
<connection url="service:jmx:rmi:///jndi/rmi://webserver1.adahas.com:9010/jmxrmi"
    username="myUser" password="myPassword" ssl="true">
```**

**Connection to a Java process running on host named webserver1.adahas.com having JMX and JNDI services running on two different ports; JMX on port 9010 and 1099. Note that standard JMX configuration will ensure that both services are available on the same port, hence this type of URL will only be needed only if JNDI service has been specifically provided on a different port.
```
<connection url="service:jmx:rmi://webserver1.adahas.com:9010/jndi/rmi://webserver1.adahas.com:1099/jmxrmi">
```**

### `<eval>` element ###

The `<eval>` element represents a single check. A check could comprise of `<query>`ing one or more JMX attributes, `<exec>`uting one or more JMX operations, performing mathematical `<expr>`ressions on the acquired values, and checking the values to be in defined warning and critical levels.

Configuration attributes available on `<eval>` element.
  * `name` - Name of the check, which will be displayed in the plugin output. This is a mandatory attribute.
  * `host` - Regular expression to perfrom on the hostname of the host before executing the check. If the pattern does not match, the check will not be executed. This would be useful if you are using a common XML file to perform multiple checks, where some of them are not applicable for some hosts/

**A simple check named 'HeapMemory'
```
<eval name="HeapMemory">
   ...
</eval>
```**

**A check named 'CacheSize' which should only be performed on hosts where the host name begins with 'cache'
```
<eval name="CacheSize" host="^cache.*">
   ...
</eval>
```**

### `<query>` element ###

### `<exec>` element ###

### `<expr>` element ###

### `<check>` element ###

### `<perf>` element ###

## Validating your configuration file ##

## Setting variables via command line to be used in the configuration file ##

# Troubleshooting #

# Examples #

## Multiple simple attribute checks (Thread count check) ##

## Using composite attributes and mathematical expressions (Heap memory usage check) ##

## Regular expression based check (Tomcat version check) ##