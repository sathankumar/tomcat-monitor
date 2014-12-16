## Monitoring Tomcat from python: a set of examples and explanations

The Tomcat server exposes a number of interfaces for monitoring and control via the [Tomcat Manager](http://tomcat.apache.org/tomcat-7.0-doc/manager-howto.html). In a lot of ways, this is excellent. It permits users with proper authorization to deploy, run, shut down, reload, and restart Tomcat applications. It also provides the ability to monitor a lot of the otherwise difficult-to-access internal statistics of apps running in the Tomcat container, and the container itself.

The Tomcat Manager is a web application that ships with Tomcat. (See the docs for how to configure it for use.) The Tomcat Manager can be used in the following ways:

 - Via the web pages built into the app.

 - Via the Text Interface, a simplistic set of web services provided mainly for scripting support.

 - Via predefined Ant commands.

 - Via JMX controls.

Of these, the simplest way to support general monitoring and automated recovery is the Text Interface. The web pages are designed for humans; Ant is a specialty item; and JMX is rapidly falling out of favor due to its problematic security characteristics. (It's not unlikely that your internal network's administrators will block JMX traffic as a matter of policy.) 

There is one odd and brutal glitch here, though. The Text Interface provides full control over deploying, starting, stopping, etc; but it's strangely scanty on monitoring. This is probably an artifact of development history, rather than an intentional move on the part of the Tomcat team. Still, it sucks, and it makes a curious hybrid approach necessary: 

 - Text Interface for everything that it _does_ support, and 

 - Screen scraping the web page that contains the monitoring info that the Text Interface will not provide.

This repo contains examples of doing this in Python. The examples try to be as simple and straightforward as possible. The current incarnation uses `requests` for HTTP client work, and `BeautifulSoup` for the screen scraping.

### Monitored statistics: Text Interface

The following information is available via the text interface (`<server>/manager/text/`):

#### Text Service: OS and JVM characteristics
URL: `<server>/manager/text/serverinfo`

This includes Tomcat version, JVM version and vendor, OS version, hostname, IP address

#### Text Service: Running Tomcat processes, by context path
URL: `<server>/manager/text/list`

For each process, lists:
 - Context Path
 - Version
 - Display Name
 - Running yes/no
 - How many user sessions

#### Text Service: JNDI Resources
URL: `<server>/manager/text/resources`

For each resource, lists JNDI name


### Monitored Statistics: scraped from status page
URL: `<server>/manager/status`

#### JVM Memory usage
This is a valuable piece of information to have: even if you don't know what these memory pools are, it's good to be able to know when they are running up against their limits.

For each pool, the page provides:
 - Memory Pool Name
 - Memory Type (heap/non-heap)
 - Initial Size
 - Total Size
 - Maximum Size
 - Currently Used (size and percentage)

The ordinary JVM memory pools are:
 - **CMS Old Gen:** The largest memory pool which should keep the long living objects. Objects are copied into this pool once they leave the survivor spaces.
 - **Par Eden Space:** The pool from which memory is initially allocated for most objects.
 - **Par Survivor Space:** The pool containing objects that have survived the garbage collection of the Eden space.
 - **CMS Perm Gen:** The pool containing all the reflective data of the virtual machine itself, such as class and method objects. With Java VMs that use class data sharing, this generation is divided into read-only and read-write areas.
 - **Code cache:** The HotSpot Java VM also includes a code cache, containing memory that is used for compilation and storage of native code.

References and content sources:
 - [stackoverflow: How is the java memory pool divided?](http://stackoverflow.com/questions/1262328/how-is-the-java-memory-pool-divided)
 - [A short Primer to Java Memory Pool Sizing and Garbage Collectors](http://www.scalingbits.com/javaprimer)
 - [Java Garbage Collection Distilled](http://www.infoq.com/articles/Java_Garbage_Collection_Distilled)


#### Connector session threads
Connectors for various protocols (HTTP, HTTPS, AJP, etc.) show some of the most important information for determining the state of an application. The Tomcat Manager provides a rich set of data.

For each thread associated with a connector, the Tomcat Manager tells us:
 - Stage: the part of the job cycle in which the thread currently resides.
   - S: Service (thread is busy satisfying a request)
   - F: Finishing (thread is done with work and is releasing resources)
   - R: Ready (thread is idle, awaiting assignment to a request)
   - K: Keepalive (thread is being used by a connection that is being held open even when there is no immediate work)
 - Time: the length of time in ms that the thread has been in this stage. Long times, especially when associated with a lack of activity, can indicate an orphaned or stuck thread.
 - Bytes Sent
 - Bytes Received
 - Client (Forwarded)
 - Client (Actual)
 - VHost
 - Request
