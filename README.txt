
-  What is netrun(1)?

   Basically, a more sophisticated, parallel version of
   the following idiom:

      $ for i in `cat hosts.lst`
        do
           ssh $i 'one or more Unix command-lines' 2> $i.err > $i.out
           echo "$i exited with $?"
        done
  
-  Quick Review of netrun(1)

   * How netrun(1) Works
   * Command-line Summary
   * Modes of Operation
   * File Transfer Hacks
   * Using Data from STDIN
   * Summary

-  How netrun(1) Works

   * Requires SSH keys and ssh-agent
   * Executes an interpreter on each host
   * Feeds commands, scripts, and data to STDIN of the interpreter
     (in that order)

-  Command-line Summary

   $ netrun -h
   Usage: netrun [-hqR] [-c connect timeout] [-f max forks]
                 [-i interpreter] [-s script file] | [-e script]
                 [-d data file] [-l login name] [-L log dir]
                 [-t timeout] hosts ...

                   -h : display help
                   -q : quick mode
                   -R : don't randomize the hosts list
           -c timeout : the SSH connect timeout (default is 15 seconds)
         -f max forks : maximum number of background processes (default is 25)
       -i interpreter : use specified interpreter instead of ksh
       -s script file : local file containing script to run on remote hosts
            -e script : like -s, but provide the script on the command-line
         -d data file : append specified data file to script
        -l login name : specify the user to login as on the remote hosts
           -L log dir : create log files in log dir instead of ./netrun.PID
           -t timeout : kill remote script after timeout seconds

-  The man(1) page for netrun(1)

   NAME
       netrun - run a script over multiple hosts in parallel

   SYNOPSIS
       netrun [-hqR]] [-f *max forks*] [-i *interpreter*] [-s *script file*] |
       [-e *script*] [-d *data file*] [-l *login name*] [-L *log dir*] [-t
       *timeout*] hosts ...

   DESCRIPTION
       Netrun provides a convenient and efficient way to run a single command
       or a script on a bunch of remote hosts. Netrun captures the output and
       error messages from the command or script for reporting and examination.

       Netrun is powered by "ssh" and assumes that you have a setup like the
       following:

       *   You have created an SSH public/private key pair using "ssh-keygen".

       *   You have copied your public key(s) to "$HOME/.ssh/authorized_keys"
           on all of the remote hosts on which you plan to run commands or
   ...
   EXAMPLES
       Tell "inetd" to re-read its configuration file on a bunch of Solaris
       hosts:
   ...

-  Standard Mode

   Example:  Grab a copy of /etc/passwd from all systems.

   $ netrun -L passwd.d -e 'cat /etc/passwd' `cat hosts.lst`
   Progress:  0% |-----------+-----------+-----------+-----------| 100%
                 ################################################# Done.

   Name/Address        Exit   Runtime  # Lines  First Line of Output
   ---------------------------------------------------------------------------
   cno8w11             no interpreter:  connect: timeout
   gen8cache14         no interpreter:  connect: timeout
   ..
   colombia               0     1.349       13  root:x:0:1:Super-User:/:/sbin/
   latitude               0     1.449       13  root:x:0:1:Super-User:/:/sbin/
   ...
   int8envoy1           127     1.160        0
   cs8s05b              127     2.040        0
   ...
   cs8s09e              255     2.195        0
   cs8s13b              255     2.207        0
   ---------------------------------------------------------------------------

   Results can be found in passwd.d.

-  Quick Mode

   Example:  Find systems where David Snyder does not have a home directory.

   $ awk '{print $2}' ~/cvs/etc/managed_hosts.lst > hosts.lst
   $ netrun -qe '[ -d ~dsnyder ] || echo David is homeless.' `cat hosts.lst`
   Progress:  0% |-----------+-----------+-----------+-----------| 100%
                 ################################################# Done.
   ripsaw: David is homeless.

-  Perl Mode

   Example:  Same as above, but in Perl.

   $ netrun -i perl -qe '
      print "David is homeless.\n" unless -d (getpwnam "dsnyder")[7]
   ' `cat hosts.lst`
   Progress:  0% |-----------+-----------+-----------+-----------| 100%
                 ################################################# Done.
   ripsaw: David is homeless.

-  File Transfer Mode

   Example:  Update your shell environment and other dot-files on all hosts.

   $ tar cf dots.tar .Xdefaults .profile .exrc .ssh/authorized_keys
   $ netrun -q -s dots.tar -i 'tar xvf -' `cat hosts.lst`
   Progress:  0% |-----------+-----------+-----------+-----------| 100%
                 ################################################# Done.
   ...
   cnn8pa2: .Xdefaults
   cnn8pa2: .profile
   cnn8pa2: .exrc
   cnn8pa2: .ssh/authorized_keys
   ...

-  Scripts with Data Files

   Example:  Copy a new rsync start config to /etc/xinetd.d and HUP xinetd.

   $ netrun -d rsync -i '
     cd /etc/xinetd.d || exit  # Skip non-xinetd systems
     rm -f rsync.bak
     mv rsync rsync.bak
     cat > rsync
     killall -HUP xinetd
   ' `cat hosts.lst`


   Example:  Add 10.188.32.119 to hosts.allow.

   $ echo "ALL: 10.188.32.119" > allow.add
   $ netrun -q -l root -i 'allow_filter -ji' -d allow.add host1 ...

-  Taking Data From Standard Input

   Example:  Guess which root passwd each host currently has.

   $ netrun -q -l root -i perl -s check_rootpwd.pl -d - `cat hosts.lst` | colfmt
   Gathering DATA from Standard Input:
   __DATA__
   External    ???????
   Internal    #######
   Webfarm     @@@@@@@
   NFS         !!!!!!!
   CTRL-D
   Progress:  0% |-----------+-----------+-----------+-----------| 100%
                 ################################################# Done.
   web:         Webfarm
   cnn8pa2:     External
   jcmsdev1:    Internal
   wfgold:      NFS
   bcpdulles1:  NOT_FOUND

-  File Transfer/Standard Input Mode, part 2

   Example:  Update hosts.allow on all un-Unified hosts.

   $ echo "ALL:  10.165.244.0/27" | \
   $ netrun -q -d - -i '/opt/wf/bin/ip-renum -ji' `cat non-unified.lst`
   Progress:  0% |-----------+-----------+-----------+-----------| 100%
                 ################################################# Done.
   ...

-  Summary

   * netrun(1) requires SSH keys and an ssh-agent
   * Feeds commands, scripts, and data to an interpreter
   * Supports Quick, Standard, and File Transfer modes
   * Can replicate data from STDIN to many hosts
   * Can save you lots of time if used correctly

