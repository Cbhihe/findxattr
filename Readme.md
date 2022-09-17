This code and the code contained in the original repo
(https://www.github.com/Cbhihe/findxattr) is protected by the terms and
conditions of the GNU_GPL-v3 copyleft license.

Copyright (C) 2021, 2022 Cedric Bhihe

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation, either version 3 of the License, or (at your option) any later
version.

This program is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
FITNESS FOR ANY PARTICULAR PURPOSE.  See the GNU General Public License for
more details.

You should have received a copy of the GNU General Public License along with
this program.  If not, see <https://www.gnu.org/licenses/>, or consult the
License.md file in this repo.

To contact the author(s), use cedric dot bhihe at gmail dot com.

---

### Organization of this git repo

There are two main branches: 'main' and 'develop'
* `main` or `master` only contains production-ready code,
* `develop` may contains other elements of code not ready yet for production or ready but not yet merged in the main branch. Use that branch code at your own risk.

In order to run `findxattr`, you should have a bash or Bourne shell and a terminal. This is the only requirement.  The code was written to be POSIX compliant and should therefore be portable although there is no guarantee that it will behave so on your particular host. 
The code was tested with Bash 5.0.x. and later. It was not actually tested on a "true" Bourne shell as found on Unices such as the Solaris OS.  Note that the Bourne shell is proprietary (not open-source) and is hard to come by those days unless you can get your hands on an OS directly derived from the origianl AT&T Unix.

### Motivation for this code
Some people like it (or need it) "hot". For instance they may have special requirements when using their file system and need to work with extended file attributes (`xattr`). The `xattr` man page defines extended  attributes as 

- name:value pairs associated permanently with files and directories, similar to the environment strings associated with a process.  An attribute may be defined or undefined.  If it is defined, its value may be empty or non-empty.
- extensions to the normal attributes which are associated with all inodes in the system (i.e., the stat(2) data).  They are often used to provide additional functionality to  a  filesystem—for example,  additional security features such as Access Control Lists (ACLs) may be implemented using extended attributes. 

When extended attributes are used, a new problem often arises: how does one seek files based on their extended attributes ? Is there an indexing facility that allows file search based on their xattr the way `mlocate` based on `updatedb` for regular attrbutes ? 

I don't know of such facility on Linux (Debian-flavored ) or BSD hosts, but on a system where xattr is enabled, rolling your own isearch tool is not overly complicated.  This repo provides a working solution in the form of a POSIX compliant script for searching (not for indexing per se). I integrated a CLI usage help functionality, so getting to the stage of using it should not require poring over every detail of the code, although "read the code" is the best advice overall.

As a final note, indexing would require running the script unchanged but as a daemon, perhaps as a systemd service unit, and updating a small database (which implies persisting pertinent data in a table of sorts). Doing so on a continuous basis however, would place a significant burden on the indexed FS's host, as in Baloo's case on KDE for instance. Short of a compelling reason to do so, I would avoid it altogether and stick to a much lighter-weight CLI spot-search utility instead, such as the one proposed here.

### Script description
`findxattr`
- is based on the well known find external utility and its idiom:<BR>

    find [options, filters ...] -exec sh -c '[...]' sh_exec "{}" \; 2> dev/null

- runs in POSIX-compliant mode even on hosts where the shebang #!/usr/bin/sh points to a fake Bourne shell, meaning that you will often find that

    /usr/bin/sh --> /usr/bin/bash.

- consists of a find-wrapper. It does not implement every bells and whistles commonly available in find implementations, but it complies with OP's requirements and goes beyond to offer a little more flexibility to ClI users interested in refining their searches. In particular, it implements:
    - a "help" mode, by issuing findxattr -h|--help, that documents correct usage,
    - long and short options, preceded respectively by one and two hyphens,
    - two filters -p|--path and -m|--md|--maxdepth, very similar to

    find [-maxdepth <d>] [-path <path_specs>] ... (see man find)

    - global searches from $PWD (when invoked with no argument, or with -a|--all),
    - tagged searches by x-attribute's KV properties, namely:
        - key -x|--xattr <xattr_name>==,
        - value -x|--xattr ==<xattr_value>,
        - both -x|--xattr <xattr_name>==<xattr_value>,
    where x-attributes' names and values can include spaces if adequately quoted.

- can be trivially expanded to include more options and switches that getopt (in the script) will gladly parse. To exploit additional options and filters, expanded logic is needed, but the way existing options are treated is a blueprint for expanding the tool's scope.

### Testing
In order to test-run the scripts, open a terminal session first to build some toy x-attributes.

    $ touch ~/fully/qualified/foo ~/fully/qualified/_1_path/bar ~/fully/qualified/_2_path/foobar ~/fully/qualified/baz
    $ setfattr -n user.likes -v 'I like strawberries.' ~/fully/qualified/foo

In the above, with the `setfattr` cmd, `user.__` points toward the user namespace for the x-attribute named "likes". New x-attributes are defined as key-value (KV) pairs. Here keys are the strings "likes", "dislikes", "filebirth" and their values must be preceded by -v.

Using the `attr`cmd instead of `setfattr` (and replacing `-v` by `-V`), x-attributes are always created in user namespace by default:

    $ cd ~/fully/qualified
    $ attr -s dislikes -V 'hacks' ./foo
    $ attr -s filebirth -V '20220627-193029-CEST' ./_1_path/bar
    $ attr -s filebirth -V '20210330-185430-CEST' ./_2_path/foobar
    $ attr -s dislikes -V 'java' ./baz

To set, get, remove or list extended attributes, you can also use attr on compatible filesystem objects (see man attr):

Usage: <BR>
    attr [-LRSq] -s attrname [-V attrvalue] pathname # to set value
    attr [-LRSq] -g attrname pathname # to get value
    attr [-LRSq] -r attrname pathname # to remove attr
    attr [-LRq] -l pathname # to list attrs
    -s reads a value from stdin and -g writes a value to stdout

So, for instance:

    $ cd ~/fully/qualified
    $ attr -qg likes ./foo
    I like strawberries.
    $ attr -qg filebirth ./_1_path/bar
    20220627-193029-CEST

The script produces a tab-separated output, e.g.:

    $ cd ~; pwd
    /home/USER
    $ findxattr -m 4                             # search entire subtree (from `$PWD`) with max depth of 4
    ./fully/qualified/foo        likes        I like strawberries.
    ./fully/qualified/foo        dislikes     hacks
    ./fully/qualified/_1_path/bar           filebirth        20220627-193029-CEST
    ./fully/qualified/_2_path/foobar        filebirth        20210330-185430-CEST
    ./fully/qualified/baz        dislikes     java
    
    $ findxattr -m 3 -x dislikes==               # search entire subtree (from `$PWD`) by name, depth
    ./fully/qualified/foo        dislikes     hacks
    ./fully/qualified/baz        dislikes     java
    
    $ findxattr -m 4 -p "*lified/_2_*" -x filebirth== # search by depth, path, name
    ./fully/qualified/_2_path/foobar        filebirth        20210330-185430-CEST
    
    $ findxattr -m 4 -x =='20220627-193029-CEST' # search entire subtree (from `$PWD`) by value, depth
    ./fully/qualified/_1_path/bar        filebirth        20220627-193029-CEST
    
    $ findxattr -x filebirth==                   # search entire subtree (from `$PWD`) by name
    ./fully/qualified/_1_path/bar        filebirth        20220627-193029-CEST
    ./fully/qualified/_2_path/foobar     filebirth        20210330-185430-CEST

### Issues
    If you experience issues with this scripts,i do not write an email to me. Instead,  please, open an issue in this Gibthub repo.
