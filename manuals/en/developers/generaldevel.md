%Bareos Developer Guide

Bareos Developer Notes
======================

This document is based on the Bacula Developer Guide
and currently only slightly adapted to Bareos.

It is planed, to update it step by step in the upcoming months.

This document is intended mostly for developers and describes how you
can contribute to the Bareos project and the the general framework of
making Bareos source changes.

### Contributions

Contributions to the Bareos project come in many forms: ideas,
participation in helping people on the bareos-users email list,
packaging Bareos binaries for the community, helping improve the
documentation, and submitting code.

### Patches

Subject to the copyright assignment described below, your patches should
be sent in <span>**git format-patch**</span> format relative to the
current contents of the master branch of the Git
repository. Please attach the output file or files generated by the
<span>**git format-patch**</span> to the email rather than include them
direct to avoid wrapping of the lines in the patch. Please be sure to
use the Bareos indenting standard (see below) for source code. If you
have checked out the source with Git, you can get a diff using.

    git pull
    git format-patch -M

If you plan on doing significant development work over a period of time,
after having your first patch reviewed and approved, you will be
eligible for having developer Git write access so that you can commit
your changes directly to the Git repository. To do so, you will need a
userid on Source Forge.


Developing Bareos
-----------------

### Debugging

Probably the first thing to do is to turn on debug output.

A good place to start is with a debug level of 20 as in
<span>**./startit -d20**</span>. The startit command starts all the
daemons with the same debug level. Alternatively, you can start the
appropriate daemon with the debug level you want. If you really need
more info, a debug level of 60 is not bad, and for just about everything
a level of 200.

### Using a Debugger

If you have a serious problem such as a segmentation fault, it can
usually be found quickly using a good multiple thread debugger such as
<span>**gdb**</span>. For example, suppose you get a segmentation
violation in <span>**bareos-dir**</span>. You might use the following to
find the problem:

<span>\<</span>start the Storage and File daemons<span>\></span> cd dird
gdb ./bareos-dir run -f -s -c ./dird.conf <span>\<</span>it dies with a
segmentation fault<span>\></span> where The <span>**-f**</span> option
is specified on the <span>**run**</span> command to inhibit
<span>**dird**</span> from going into the background. You may also want
to add the <span>**-s**</span> option to the run command to disable
signals which can potentially interfere with the debugging.

As an alternative to using the debugger, each <span>**Bareos**</span>
daemon has a built in back trace feature when a serious error is
encountered. It calls the debugger on itself, produces a back trace, and
emails the report to the developer. For more details on this, please see
the chapter in the main Bareos manual entitled “What To Do When Bareos
Crashes (Kaboom)”.

### Memory Leaks

Because Bareos runs routinely and unattended on client and server
machines, it may run for a long time. As a consequence, from the very
beginning, Bareos uses SmartAlloc to ensure that there are no memory
leaks. To make detection of memory leaks effective, all Bareos code that
dynamically allocates memory MUST have a way to release it. In general
when the memory is no longer needed, it should be immediately released,
but in some cases, the memory will be held during the entire time that
Bareos is executing. In that case, there MUST be a routine that can be
called at termination time that releases the memory. In this way, we
will be able to detect memory leaks. Be sure to immediately correct any
and all memory leaks that are printed at the termination of the daemons.

### When Implementing Incomplete Code

Please identify all incomplete code with a comment that contains

    ***FIXME***

where there are three asterisks (\*) before and after the word FIXME (in
capitals) and no intervening spaces. This is important as it allows new
programmers to easily recognize where things are partially implemented.


### Header Files

Please carefully follow the scheme defined below as it permits in
general only two header file includes per C file, and thus vastly
simplifies programming. With a large complex project like Bareos, it
isn’t always easy to ensure that the right headers are invoked in the
right order (there are a few kludges to make this happen – i.e. in a few
include files because of the chicken and egg problem, certain references
to typedefs had to be replaced with <span>**void**</span> ).

Every file should include <span>**bareos.h**</span>. It pulls in just
about everything, with very few exceptions. If you have system dependent
ifdefing, please do it in <span>**baconfig.h**</span>. The version
number and date are kept in <span>**version.h**</span>.

Each of the subdirectories (console, cats, dird, filed, findlib, lib,
stored, ...) contains a single directory dependent include file
generally the name of the directory, which should be included just after
the include of <span>**bareos.h**</span>. This file (for example, for
the dird directory, it is <span>**dird.h**</span>) contains either
definitions of things generally needed in this directory, or it includes
the appropriate header files. It always includes
<span>**protos.h**</span>. See below.

Each subdirectory contains a header file named
<span>**protos.h**</span>, which contains the prototypes for subroutines
exported by files in that directory. <span>**protos.h**</span> is always
included by the main directory dependent include file.

### Programming Standards

For the most part, all code should be written in C unless there is a
burning reason to use C++, and then only the simplest C++ constructs
will be used. Note, Bareos is slowly evolving to use more and more C++.

Code should have some documentation – not a lot, but enough so that I
can understand it. Look at the current code, and you will see that I
document more than most, but am definitely not a fanatic.

We prefer simple linear code where possible. Gotos are strongly
discouraged except for handling an error to either bail out or to retry
some code, and such use of gotos can vastly simplify the program.

Remember this is a C program that is migrating to a
<span>**tiny**</span> subset of C++, so be conservative in your use of
C++ features.

### Do Not Use

-   STL – it is totally incomprehensible.

### Avoid if Possible

-   Using <span>**void \***</span> because this generally means that one
    must using casting, and in C++ casting is rather ugly. It is OK to
    use void \* to pass structure address where the structure is not
    known to the routines accepting the packet (typically callback
    routines). However, declaring “void \*buf” is a bad idea. Please use
    the correct types whenever possible.

-   Using undefined storage specifications such as (short, int, long,
    long long, size\_t ...). The problem with all these is that the
    number of bytes they allocate depends on the compiler and the
    system. Instead use Bareos’s types (int8\_t, uint8\_t, int32\_t,
    uint32\_t, int64\_t, and uint64\_t). This guarantees that the
    variables are given exactly the size you want. Please try at all
    possible to avoid using size\_t ssize\_t and the such. They are very
    system dependent. However, some system routines may need them, so
    their use is often unavoidable.

-   Returning a malloc’ed buffer from a subroutine – someone will forget
    to release it.

-   Heap allocation (malloc) unless needed – it is expensive. Use
    POOL\_MEM instead.

-   Templates – they can create portability problems.

-   Fancy or tricky C or C++ code, unless you give a good explanation of
    why you used it.

-   Too much inheritance – it can complicate the code, and make reading
    it difficult (unless you are in love with colons)

### Do Use Whenever Possible

-   Locking and unlocking within a single subroutine.

-   A single point of exit from all subroutines. A goto is perfectly OK
    to use to get out early, but only to a label named bail\_out, and
    possibly an ok\_out. See current code examples.

-   malloc and free within a single subroutine.

-   Comments and global explanations on what your code or algorithm
    does.

-   When committing a fix for a bug, make the comment of the following
    form:

        Fixes #1234: Description of the bug.

        Reason for bug fix
        or other message.

    It is important to write the <span>**bug \#1234**</span> like that
    because our program that automatically pulls messages from the git
    repository to make the ChangeLog looks for that pattern. Obviously
    the <span>**1234**</span> should be replaced with the number of the
    bug you actually fixed.

    Providing the commit comment line has one of the following keywords
    (or phrases), it will be ignored:

          tweak
          typo
          cleanup
          bweb:
          regress:
          again
          .gitignore
          fix compilation
          technotes
          update version
          update technotes
          update projects
          update releasenotes
          update version
          update home
          update release
          update todo
          update notes
          update changelog

-   Use the following keywords at the beginning of a git commit message

### Indenting Standards

We find it very hard to read code indented 8 columns at a time. Even 4
at a time uses a lot of space, so we have adopted indenting 3 spaces at
every level. Note, indention is the visual appearance of the source on
the page, while tabbing is replacing a series of up to 8 spaces from a
tab character.

The closest set of parameters for the Linux <span>**indent**</span>
program that will produce reasonably indented code are:

    -nbad -bap -bbo -nbc -br -brs -c36 -cd36 -ncdb -ce -ci3 -cli0
    -cp36 -d0 -di1 -ndj -nfc1 -nfca -hnl -i3 -ip0 -l85 -lp -npcs
    -nprs -npsl -saf -sai -saw -nsob -nss -nbc -ncs -nbfda

You can put the above in your .indent.pro file, and then just invoke
indent on your file. However, be warned. This does not produce perfect
indenting, and it will mess up C++ class statements pretty badly.

Braces are required in all if statements (missing in some very old
code). To avoid generating too many lines, the first brace appears on
the first line (e.g. of an if), and the closing brace is on a line by
itself. E.g.

       if (abc) {
          some_code;
      }

Just follow the convention in the code. For example we I prefer
non-indented cases.

       switch (code) {
       case 'A':
          do something
          break;
       case 'B':
          again();
          break;
       default:
          break;
      }

Avoid using // style comments except for temporary code or turning off
debug code. Standard C comments are preferred (this also keeps the code
closer to C).

Attempt to keep all lines less than 85 characters long so that the whole
line of code is readable at one time. This is not a rigid requirement.

Always put a brief description at the top of any new file created
describing what it does and including your name and the date it was
first written. Please don’t forget any Copyrights and acknowledgments if
it isn’t 100% your code. Also, include the Bareos copyright notice that
is in <span>**src/c**</span>.

In general you should have two includes at the top of the an include for
the particular directory the code is in, for includes are needed, but
this should be rare.

In general (except for self-contained packages), prototypes should all
be put in <span>**protos.h**</span> in each directory.

Always put space around assignment and comparison operators.

       a = 1;
       if (b >= 2) {
         cleanup();
      }

but your can compress things in a <span>**for**</span> statement:

       for (i=0; i < del.num_ids; i++) {
        ...

Don’t overuse the inline if (?:). A full <span>**if**</span> is
preferred, except in a print statement, e.g.:

       if (ua->verbose \&& del.num_del != 0) {
          bsendmsg(ua, _("Pruned %d %s on Volume %s from catalog.\n"), del.num_del,
             del.num_del == 1 ? "Job" : "Jobs", mr->VolumeName);
      }

Leave a certain amount of debug code (Dmsg) in code you submit, so that
future problems can be identified. This is particularly true for
complicated code likely to break. However, try to keep the debug code to
a minimum to avoid bloating the program and above all to keep the code
readable.

Please keep the same style in all new code you develop. If you include
code previously written, you have the option of leaving it with the old
indenting or re-indenting it. If the old code is indented with 8 spaces,
then please re-indent it to Bareos standards.

If you are using <span>**vim**</span>, simply set your tabstop to 8 and
your shiftwidth to 3.

### Tabbing

Tabbing (inserting the tab character in place of spaces) is as normal on
all Unix systems – a tab is converted space up to the next column
multiple of 8. My editor converts strings of spaces to tabs
automatically – this results in significant compression of the files.
Thus, you can remove tabs by replacing them with spaces if you wish.
Please don’t confuse tabbing (use of tab characters) with indenting
(visual alignment of the code).

### Don’ts

Please don’t use:

    strcpy()
    strcat()
    strncpy()
    strncat();
    sprintf()
    snprintf()

They are system dependent and un-safe. These should be replaced by the
Bareos safe equivalents:

    char *bstrncpy(char *dest, char *source, int dest_size);
    char *bstrncat(char *dest, char *source, int dest_size);
    int bsnprintf(char *buf, int32_t buf_len, const char *fmt, ...);
    int bvsnprintf(char *str, int32_t size, const char  *format, va_list ap);

See src/lib/bsys.c for more details on these routines.

Don’t use the <span>**%lld**</span> or the <span>**%q**</span> printf
format editing types to edit 64 bit integers – they are not portable.
Instead, use <span>**%s**</span> with <span>**edit\_uint64()**</span>.
For example:

       char buf[100];
       uint64_t num = something;
       char ed1[50];
       bsnprintf(buf, sizeof(buf), "Num=%s\n", edit_uint64(num, ed1));

Note: <span>**%lld**</span> is now permitted in Bareos code – we have
our own printf routines which handle it correctly. The edit\_uint64()
subroutine can still be used if you wish, but over time, most of that
old style will be removed.

The edit buffer <span>**ed1**</span> must be at least 27 bytes long to
avoid overflow. See src/lib/edit.c for more details. If you look at the
code, don’t start screaming that I use <span>**lld**</span>. I actually
use subtle trick taught to me by John Walker. The <span>**lld**</span>
that appears in the editing routine is actually
<span>**\#define**</span> to a what is needed on your OS (usually “lld”
or “q”) and is defined in autoconf/configure.in for each OS. C string
concatenation causes the appropriate string to be concatenated to the
“%”.

Also please don’t use the STL or Templates or any complicated C++ code.

### Message Classes

Currently, there are five classes of messages: Debug, Error, Job,
Memory, and Queued.

### Debug Messages

Debug messages are designed to be turned on at a specified debug level
and are always sent to STDOUT. There are designed to only be used in the
development debug process. They are coded as:

DmsgN(level, message, arg1, ...) where the N is a number indicating how
many arguments are to be substituted into the message (i.e. it is a
count of the number arguments you have in your message – generally the
number of percent signs (%)). <span>**level**</span> is the debug level
at which you wish the message to be printed. message is the debug
message to be printed, and arg1, ... are the arguments to be
substituted. Since not all compilers support \#defines with varargs, you
must explicitly specify how many arguments you have.

When the debug message is printed, it will automatically be prefixed by
the name of the daemon which is running, the filename where the Dmsg is,
and the line number within the file.

Some actual examples are:

Dmsg2(20, “MD5len=%d MD5=%s\\n”, strlen(buf), buf);

Dmsg1(9, “Created client %s record\\n”, client-\>hdr.name);

### Error Messages

Error messages are messages that are related to the daemon as a whole
rather than a particular job. For example, an out of memory condition my
generate an error message. They should be very rarely needed. In
general, you should be using Job and Job Queued messages (Jmsg and
Qmsg). They are coded as:

EmsgN(error-code, level, message, arg1, ...) As with debug messages, you
must explicitly code the of arguments to be substituted in the message.
error-code indicates the severity or class of error, and it may be one
of the following:

<span>lp<span>3in</span></span> <span><span>**M\_ABORT**</span> </span>
& <span>Causes the daemon to immediately abort. This should be used only
in extreme cases. It attempts to produce a traceback. </span>\
<span><span>**M\_ERROR\_TERM**</span> </span> & <span>Causes the daemon
to immediately terminate. This should be used only in extreme cases. It
does not produce a traceback. </span>\
<span><span>**M\_FATAL**</span> </span> & <span>Causes the daemon to
terminate the current job, but the daemon keeps running </span>\
<span><span>**M\_ERROR**</span> </span> & <span>Reports the error. The
daemon and the job continue running </span>\
<span><span>**M\_WARNING**</span> </span> & <span>Reports an warning
message. The daemon and the job continue running </span>\
<span><span>**M\_INFO**</span> </span> & <span>Reports an informational
message.</span>

There are other error message classes, but they are in a state of being
redesigned or deprecated, so please do not use them. Some actual
examples are:

Emsg1(M\_ABORT, 0, “Cannot create message thread: %s\\n”,
strerror(status));

Emsg3(M\_WARNING, 0, “Connect to File daemon %s at %s:%d failed.
Retrying ...\\n”, client-<span>\></span>hdr.name,
client-<span>\></span>address, client-<span>\></span>port);

Emsg3(M\_FATAL, 0, “bdird<span>\<</span>filed: bad response from Filed
to %s command: %d %s\\n”, cmd, n, strerror(errno));

### Job Messages

Job messages are messages that pertain to a particular job such as a
file that could not be saved, or the number of files and bytes that were
saved. They Are coded as:

    Jmsg(jcr, M\_FATAL, 0, "Text of message");

A Jmsg with M\_FATAL will fail the job. The Jmsg() takes varargs so can
have any number of arguments for substituted in a printf like format.
Output from the Jmsg() will go to the Job report. \<br\> If the Jmsg is
followed with a number such as Jmsg1(...), the number indicates the
number of arguments to be substituted (varargs is not standard for
\#defines), and what is more important is that the file and line number
will be prefixed to the message. This permits a sort of debug from
user’s output.

### Queued Job Messages

Queued Job messages are similar to Jmsg()s except that the message is
Queued rather than immediately dispatched. This is necessary within the
network subroutines and in the message editing routines. This is to
prevent recursive loops, and to ensure that messages can be delivered
even in the event of a network error.

### Memory Messages

Memory messages are messages that are edited into a memory buffer.
Generally they are used in low level routines such as the low level
device file dev.c in the Storage daemon or in the low level Catalog
routines. These routines do not generally have access to the Job Control
Record and so they return error essages reformatted in a memory buffer.
Mmsg() is the way to do this.

### Bugs Database

We have a bugs database which is at https://bugs.bareos.org.

The first thing is if you want to take over a bug, rather than just make
a note, you should assign the bug to yourself. This helps other
developers know that you are the principal person to deal with the bug.
You can do so by going into the bug and clicking on the <span>**Update
Issue**</span> button. Then you simply go to the <span>**Assigned
To**</span> box and select your name from the drop down box. To actually
update it you must click on the <span>**Update Information**</span>
button a bit further down on the screen, but if you have other things to
do such as add a Note, you might wait before clicking on the
<span>**Update Information**</span> button.

Generally, we set the <span>**Status**</span> field to either
acknowledged, confirmed, or feedback when we first start working on the
bug. Feedback is set when we expect that the user should give us more
information.

Normally, once you are reasonably sure that the bug is fixed, and a
patch is made and attached to the bug report, and/or in the SVN, you can
close the bug. If you want the user to test the patch, then leave the
bug open, otherwise close it and set <span>**Resolution**</span> to
<span>**Fixed**</span>. We generally close bug reports rather quickly,
even without confirmation, especially if we have run tests and can see
that for us the problem is fixed. However, in doing so, it avoids
misunderstandings if you leave a note while you are closing the bug that
says something to the following effect: We are closing this bug because
... If for some reason, it does not fix your problem, please feel free
to reopen it, or to open a new bug report describing the problem".

We do not recommend that you attempt to edit any of the bug notes that
have been submitted, nor to delete them or make them private. In fact,
if someone accidentally makes a bug note private, you should ask the
reason and if at all possible (with his agreement) make the bug note
public.

If the user has not properly filled in most of the important fields
(platorm, OS, Product Version, ...) please do not hesitate to politely
ask him. Also, if the bug report is a request for a new feature, please
politely send the user to the Feature Request menu item on
www.bareos.org. The same applies to a support request (we answer only
bugs), you might give the user a tip, but please politely refer him to
the manual and the Getting Support page of www.bareos.org.