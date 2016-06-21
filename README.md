# HelloWorldAnt
Repositorio con ejemplo de Hello World desplegado con Ant

Preparing the project

We want to separate the source from the generated files, so our java source files will be in src folder. All generated files should be under build, and there splitted into several subdirectories for the individual steps: classes for our compiled files and jar for our own JAR-file.

We have to create only the src directory. (Because I am working on Windows, here is the win-syntax - translate to your shell):

md src

The following simple Java class just prints a fixed message out to STDOUT, so just write this code into src\oata\HelloWorld.java.
package oata;

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}

Now just try to compile and run that:
md build\classes
javac -sourcepath src -d build\classes src\oata\HelloWorld.java
java -cp build\classes oata.HelloWorld

which will result inHello World

Creating a jar-file is not very difficult. But creating a startable jar-file needs more steps: create a manifest-file containing the start class, creating the target directory and archiving the files.
echo Main-Class: oata.HelloWorld>myManifest
md build\jar
jar cfm build\jar\HelloWorld.jar myManifest -C build\classes .
java -jar build\jar\HelloWorld.jar

Note: Do not have blanks around the >-sign in the echo Main-Class instruction because it would falsify it!

Four steps to a running applicationAfter finishing the java-only step we have to think about our build process. We have to compile our code, otherwise we couldn't start the program. Oh - "start" - yes, we could provide a target for that. We should package our application. Now it's only one class - but if you want to provide a download, no one would download several hundreds files ... (think about a complex Swing GUI - so let us create a jar file. A startable jar file would be nice ... And it's a good practise to have a "clean" target, which deletes all the generated stuff. Many failures could be solved just by a "clean build".

By default Ant uses build.xml as the name for a buildfile, so our .\build.xml would be:

<project>

    <target name="clean">
        <delete dir="build"/>
    </target>

    <target name="compile">
        <mkdir dir="build/classes"/>
        <javac srcdir="src" destdir="build/classes"/>
    </target>

    <target name="jar">
        <mkdir dir="build/jar"/>
        <jar destfile="build/jar/HelloWorld.jar" basedir="build/classes">
            <manifest>
                <attribute name="Main-Class" value="oata.HelloWorld"/>
            </manifest>
        </jar>
    </target>

    <target name="run">
        <java jar="build/jar/HelloWorld.jar" fork="true"/>
    </target>

</project>

Now you can compile, package and run the application via
ant compile
ant jar
ant run

Or shorter with
ant compile jar run

While having a look at the buildfile, we will see some similar steps between Ant and the java-only commands:
java-onlyAnt
md build\classes
javac
    -sourcepath src
    -d build\classes
    src\oata\HelloWorld.java
echo Main-Class: oata.HelloWorld>mf
md build\jar
jar cfm
    build\jar\HelloWorld.jar
    mf
    -C build\classes
    .



java -jar build\jar\HelloWorld.jar
  

<mkdir dir="build/classes"/>
<javac
    srcdir="src"
    destdir="build/classes"/>
<!-- automatically detected --><!-- obsolete; done via manifest tag -->
<mkdir dir="build/jar"/>
<jar
    destfile="build/jar/HelloWorld.jar"

    basedir="build/classes">
    <manifest>
        <attribute name="Main-Class" value="oata.HelloWorld"/>
    </manifest>
</jar>
<java jar="build/jar/HelloWorld.jar" fork="true"/>
  

Enhance the build fileNow we have a working buildfile we could do some enhancements: many time you are referencing the same directories, main-class and jar-name are hard coded, and while invocation you have to remember the right order of build steps.
The first and second point would be addressed with properties, the third with a special property - an attribute of the <project>-tag and the fourth problem can be solved using dependencies.
<project name="HelloWorld" basedir="." default="main">

    <property name="src.dir"     value="src"/>

    <property name="build.dir"   value="build"/>
    <property name="classes.dir" value="${build.dir}/classes"/>
    <property name="jar.dir"     value="${build.dir}/jar"/>

    <property name="main-class"  value="oata.HelloWorld"/>



    <target name="clean">
        <delete dir="${build.dir}"/>
    </target>

    <target name="compile">
        <mkdir dir="${classes.dir}"/>
        <javac srcdir="${src.dir}" destdir="${classes.dir}"/>
    </target>

    <target name="jar" depends="compile">
        <mkdir dir="${jar.dir}"/>
        <jar destfile="${jar.dir}/${ant.project.name}.jar" basedir="${classes.dir}">
            <manifest>
                <attribute name="Main-Class" value="${main-class}"/>
            </manifest>
        </jar>
    </target>

    <target name="run" depends="jar">
        <java jar="${jar.dir}/${ant.project.name}.jar" fork="true"/>
    </target>

    <target name="clean-build" depends="clean,jar"/>

    <target name="main" depends="clean,run"/>

</project>

Now it's easier, just do a ant and you will get
Buildfile: build.xml

clean:

compile:
    [mkdir] Created dir: C:\...\build\classes
    [javac] Compiling 1 source file to C:\...\build\classes

jar:
    [mkdir] Created dir: C:\...\build\jar
      [jar] Building jar: C:\...\build\jar\HelloWorld.jar

run:
     [java] Hello World

main:

BUILD SUCCESSFUL

Using external librariesSomehow told us not to use syso-statements. For log-Statements we should use a Logging-API - customizable on a high degree (including switching off during usual life (= not development) execution). We use Log4J for that, because

	* it is not part of the JDK (1.4+) and we want to show how to use external libs
	* it can run under JDK 1.2 (as Ant)
	* it's highly configurable
	* it's from Apache ;-)

We store our external libraries in a new directory lib. Log4J can be downloaded [1] from Logging's Homepage. Create the lib directory and extract the log4j-1.2.9.jar into that lib-directory. After that we have to modify our java source to use that library and our buildfile so that this library could be accessed during compilation and run.

Working with Log4J is documented inside its manual. Here we use the MyApp-example from the Short Manual [2]. First we have to modify the java source to use the logging framework:
package oata;

import org.apache.log4j.Logger;import org.apache.log4j.BasicConfigurator;

public class HelloWorld {
    static Logger logger = Logger.getLogger(HelloWorld.class);

    public static void main(String[] args) {
        BasicConfigurator.configure();
        logger.info("Hello World");          // the old SysO-statement
    }
}

Most of the modifications are "framework overhead" which has to be done once. The blue line is our "old System-out" statement.
Don't try to run ant - you will only get lot of compiler errors. Log4J is not inside the classpath so we have to do a little work here. But do not change the CLASSPATH environment variable! This is only for this project and maybe you would break other environments (this is one of the most famous mistakes when working with Ant). We introduce Log4J (or to be more precise: all libraries (jar-files) which are somewhere under .\lib) into our buildfile:
<project name="HelloWorld" basedir="." default="main">
    ...
    <property name="lib.dir"     value="lib"/>

    <path id="classpath">
        <fileset dir="${lib.dir}" includes="**/*.jar"/>
    </path>

    ...

    <target name="compile">
        <mkdir dir="${classes.dir}"/>
        <javac srcdir="${src.dir}" destdir="${classes.dir}" classpathref="classpath"/>
    </target>

    <target name="run" depends="jar">
        <java fork="true" classname="${main-class}">
            <classpath>
                <path refid="classpath"/>
                <path location="${jar.dir}/${ant.project.name}.jar"/>
            </classpath>
        </java>
    </target>

    ...

</project>

In this example we start our application not via its Main-Class manifest-attribute, because we could not provide a jarname and a classpath. So add our class in the red line to the already defined path and start as usual. Running ant would give (after the usual compile stuff):
[java] 0 [main] INFO oata.HelloWorld  - Hello World

What's that?

	* [java] Ant task running at the moment
	* 0 sorry don't know - some Log4J stuff
	* [main] the running thread from our application
	* INFO log level of that statement
	* oata.HelloWorld source of that statement
	* - separator
	* Hello World the message

For another layout ... have a look inside Log4J's documentation about using other PatternLayout's.Configuration filesWhy we have used Log4J? "It's highly configurable"? No - all is hard coded! But that is not the debt of Log4J - it's ours. We had coded BasicConfigurator.configure(); which implies a simple, but hard coded configuration. More comfortable would be using a property file. In the java source delete the BasicConfiguration-line from the main() method (and the related import-statement). Log4J will search then for a configuration as described in it's manual. Then create a new file src/log4j.properties. That's the default name for Log4J's configuration and using that name would make life easier - not only the framework knows what is inside, you too!

log4j.rootLogger=DEBUG, stdout

log4j.appender.stdout=org.apache.log4j.ConsoleAppender

log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%m%n
This configuration creates an output channel ("Appender") to console named as stdout which prints the message (%m) followed by a line feed (%n) - same as the earlier System.out.println() :-) Oooh kay - but we haven't finished yet. We should deliver the configuration file, too. So we change the buildfile:
    ...
    <target name="compile">
        <mkdir dir="${classes.dir}"/>
        <javac srcdir="${src.dir}" destdir="${classes.dir}" classpathref="classpath"/>
        <copy todir="${classes.dir}">
            <fileset dir="${src.dir}" excludes="**/*.java"/>
        </copy>
    </target>
    ...

This copies all resources (as long as they haven't the suffix ".java") to the build directory, so we could start the application from that directory and these files will included into the jar.
Testing the classIn this step we will introduce the usage of the JUnit [3] testframework in combination with Ant. Because Ant has a built-in JUnit 3.8.2 you could start directly using it. Write a test class in src\HelloWorldTest.java:
public class HelloWorldTest extends junit.framework.TestCase {

    public void testNothing() {
    }
    
    public void testWillAlwaysFail() {
        fail("An error message");
    }
    
}
Because we dont have real business logic to test, this test class is very small: just show how to start. For further information see the JUnit documentation [3] and the manual of junit task. Now we add a junit instruction to our buildfile:
    ...

    <path id="application" location="${jar.dir}/${ant.project.name}.jar"/>

    <target name="run" depends="jar">
        <java fork="true" classname="${main-class}">
            <classpath>
                <path refid="classpath"/>
                <path refid="application"/>
            </classpath>
        </java>
    </target>
    
    <target name="junit" depends="jar">
        <junit printsummary="yes">
            <classpath>
                <path refid="classpath"/>
                <path refid="application"/>
            </classpath>
            
            <batchtest fork="yes">
                <fileset dir="${src.dir}" includes="*Test.java"/>
            </batchtest>
        </junit>
    </target>

    ...


We reuse the path to our own jar file as defined in run-target by giving it an ID and making it globally available. The printsummary=yes lets us see more detailed information than just a "FAILED" or "PASSED" message. How much tests failed? Some errors? Printsummary lets us know. The classpath is set up to find our classes. To run tests the batchtest here is used, so you could easily add more test classes in the future just by naming them *Test.java. This is a common naming scheme.
After a ant junit you'll get:
...
junit:
    [junit] Running HelloWorldTest
    [junit] Tests run: 2, Failures: 1, Errors: 0, Time elapsed: 0,01 sec
    [junit] Test HelloWorldTest FAILED

BUILD SUCCESSFUL
...

We can also produce a report. Something that you (and other) could read after closing the shell .... There are two steps: 1. let <junit> log the information and 2. convert these to something readable (browsable).
    ...
    <property name="report.dir"  value="${build.dir}/junitreport"/>
    ...
    <target name="junit" depends="jar">
        <mkdir dir="${report.dir}"/>
        <junit printsummary="yes">
            <classpath>
                <path refid="classpath"/>
                <path refid="application"/>
            </classpath>
            
            <formatter type="xml"/>
            
            <batchtest fork="yes" todir="${report.dir}">
                <fileset dir="${src.dir}" includes="*Test.java"/>
            </batchtest>
        </junit>
    </target>
    
    <target name="junitreport">
        <junitreport todir="${report.dir}">
            <fileset dir="${report.dir}" includes="TEST-*.xml"/>
            <report todir="${report.dir}"/>
        </junitreport>
    </target>
Because we would produce a lot of files and these files would be written to the current directory by default, we define a report directory, create it before running the junit and redirect the logging to it. The log format is XML so junitreport could parse it. In a second target junitreport should create a browsable HTML-report for all generated xml-log files in the report directory. Now you can open the ${report.dir}\index.html and see the result (looks something like JavaDoc).
Personally I use two different targets for junit and junitreport. Generating the HTML report needs some time and you dont need the HTML report just for testing, e.g. if you are fixing an error or a integration server is doing a job.
Resources    [1] http://www.apache.org/dist/logging/log4j/1.2.13/logging-log4j-1.2.13.zip
    [2] http://logging.apache.org/log4j/docs/manual.html
    [3] http://www.junit.org/index.htm

