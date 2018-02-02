Introducing To Writing Wapp Applications
========================================

Wapp applications are easy to develop.  A hello-world program is as follows:

>
    #!/usr/bin/wapptclsh
    package require wapp
    proc wapp-default {req} {
       wapp-subst {<h1>Hello, World!</h1>\n}
    }
    wapp-start $::argv

Every Wapp application defines one or more procedures that accept HTTP
requests and generate appropriate replies.
For an HTTP request where the initial portion of the URI path is "abcde", the
procedure named "wapp-page-abcde" will be invoked to construct the reply.
If no such procedure exists, "wapp-default" is invoked instead.  The latter
technique is used for the hello-world example above.

The hello-world example generates the reply using a single call to the "wapp-subst"
command.  Each "wapp-subst" command appends new text to the reply, applying
various substitutions as it goes.  The only substitution in this example is
the \\n at the end of the line.

The "wapp-start" command starts up the application.

To run this application, copy the code above into a file named "main.tcl"
and then enter the following command:

>
    wapptclsh main.tcl

That command will start up a web-server bound to the loopback
IP address, then launch a web-browser pointing at that web-server.
The result is that the "Hello, World!" page will automatically
appear in your web browser.

To run this same program as a traditional web-server on TCP port 8080, enter:

>
    wapptclsh main.tcl --server 8080

Here the built-in web-server listens on all IP addresses and so the
web page is available on other machines.  But the web-broswer is not
automatically started in this case, so you will have to manually enter
"http://localhost:8080/" into your web-browser in order to see the page.

To run this program as CGI, put the main.tcl script in your web-servers
file hierarchy, in the appropriate place for CGI scripts, and make any
other web-server specific configuration changes so that the web-server
understands that the main.tcl file is a CGI script.  Then point your
web-browser at that script.

Run the hello-world program as SCGI like this:

>
    wapptclsh main.tcl --scgi 9000

Then configure your web-server to send SCGI requests to TCL port 9000
for some specific URI, and point your web-browser at that URI.

1.0 Using Plain Old Tclsh
-------------------------

Wapp applications are pure TCL code.  You can run them using an ordinary
"tclsh" command if desired, instead of the "wapptclsh" shown above.  We
normally use "wapptclsh" for the following reasons:

   +  Wapptclsh is statically linked, so there is never a worry about
      having the right shared libraries on hand.  This is particularly
      important if the application will ultimately be deployed into a
      chroot jail.

   +  Wapptclsh has SQLite built-in and SQLite turns out to be very
      useful for the kinds of small application where Wapp excels.

   +  Wapptclsh knows how to process "package require wapp".  If you
      run with ordinary tclsh, you might need to change the 
      "package require wapp" into "source wapp.tcl" and ship the separate
      "wapp.tcl" script together with your application.

We prefer to use wapptclsh and wapptclsh is shown in all of the examples.
But ordinary "tclsh" will work in the examples too.

2.0 A Slightly Longer Example
-----------------------------

Wapp keeps track of various [parameters](params.md) that describe
each HTTP request.  Those parameters are accessible using routines
like "wapp-param _NAME_"
The following sample program gives some examples:

>
    package require wapp
    proc wapp-default {} {
      set B [wapp-param BASE_URL]
      wapp-trim {
        <h1>Hello, World!</h1>
        <p>See the <a href='%html($B)/env'>Wapp
        Environment</a></p>
      }
    }
    proc wapp-page-env {} {
      wapp-allow-xorigin-params
      wapp-subst {<h1>Wapp Environment</h1>\n<pre>\n}
      foreach var [lsort [wapp-param-list]] {
        if {[string index $var 0]=="."} continue
        wapp-subst {%html($var) = %html([list [wapp-param $var]])\n}
      }
      wapp-subst {</pre>\n}
    }
    wapp-start $::argv

In this application, the default "Hello, World!" page has been extended
with a hyperlink to the /env page.  The "wapp-subst" command has been
replaced by "wapp-trim", which works the same way with the addition that
it removes surplus whitespace from the left margin, so that the generated
HTML text does not come out indented.  The "wapp-trim" and "wapp-subst"
commands in this example use "%html(...)" substitutions.  The "..." argument 
is expanded using the usual TCL rules, but then the result is escaped so
that it is safe to include in an HTML document.  Other supported
substitutions are "%url(...)" for
URLs on the href= and src= attributes of HTML entities, "%qp(...)" for
query parameters, "%string(...)" for string literals within javascript,
and "%unsafe(...)" for direct literal substitution.  As its name implies,
the %unsafe() substitution should be avoid whenever possible.

The /env page is implemented by the "wapp-page-env" proc.  This proc
generates HTML that describes all of the query parameters. Parameter names
that begin with "." are for internal use by Wapp and are skipped
for this display.  Notice the use of "wapp-subst" to safely escape text
for inclusion in an HTML document.

3.0 General Structure Of A Wapp Application
-------------------------------------------

Wapp applications all follow the same basic template:

>
    package require wapp;
    proc wapp-page-XXXXX {} {
      # code to generate page XXXX
    }
    proc wapp-page-YYYYY {} {
      # code to generate page YYYY
    }
    proc wapp-default {} {
      # code to generate any page not otherwise
      # covered by wapp-page-* procs
    }
    wapp-start $argv

The application script first loads the Wapp code itself using
the "package require" at the top.  (Some applications may choose
to substitute "source wapp.tcl" to accomplish the same thing.)
Next the application defines various procs that will generate the
replies to HTTP requests.  Differ procs are invoked based on the
first element of the URI past the Wapp script name.  Finally,
the "wapp-start" routine is called to start Wapp running.  The
"wapp-start" routine never returns, so it should be the very last
command in the application script.