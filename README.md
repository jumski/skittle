# Skittle

A tiny tool for simplifying system provisioning with simple bash scripts.

[![Build Status](https://secure.travis-ci.org/d11wtq/skittle.png?branch=master)]
(http://travis-ci.org/d11wtq/skittle)

## Overview

Skittle makes it really easy to script install procedures, configure computers
or do just about anything that takes multiple steps using bash.

## Installation

Because Skittle aims for zero dependencies, it is not intended that you install
it in your system, instead prefering that you just add it to your project
directly. It is less than 200 lines long (including whitespace and function
definitions) and lives in a single file.

``` bash
curl -O https://raw.githubusercontent.com/d11wtq/skittle/master/bin/skittle
chmod +x skittle
```

If you do want to add it to your system, go ahead place it on your `$PATH`.

## Usage

There are very few concepts to learn when it comes to using Skittle. The first
concept is knowing that the `skittle` executable takes a single argument, which
is the name of a dependency it should resolve.

The second concept, is that a dependency is just a function that defines two
other functions: one to indicate if it needs to run, and another that specifies
how to run it.

Finally, dependencies may specify other depdencies that they require to run
(i.e. they are recursively defined).

Let's look at a simple example of creating a log directory for a fictonal
service called "turtle" (It was just a name that came to mind!)

We'll run this dependency like so.

```
bash-3.2$ ./skittle log_dir
+ log_dir
|
\ [fail] log_dir
Error: Cannot find dependency 'log_dir'
```

As you might expect, this produces an error, because we haven't actually
specified what the `log_dir` dependency should do. We need to create a file
named `"./deps/log_dir.sh"` and add a small amount of code.

``` bash
# ./deps/log_dir.sh

log_dir() {
  dir_path=/var/log/turtle

  is_met() {
    ls $dir_path
  }

  meet() {
    sudo mkdir $dir_path
  }
}
```

Save the file and go ahead and run `skittle log_dir` again. You should see
everything go green and an `[ok]` indicator.

```
bash-3.2$ ./skittle log_dir
+ log_dir
|
\ [ok] log_dir
bash-3.2$
```

> **Note** `sudo` may prompt for your password when you run this dep.

If you take a look , `/var/log/turtle` should have been created.

Quite simply, Skittle looks for your depedency under `./deps`, using the
convention that it has a file name matching the dependency, but ending in
".sh".

Next, it evaluates the function—in a subshell—knowing that it produces `is_met`
and `meet` functions.

If the `is_met` function returns a zero exit status, nothing happens. If,
however it returns non-zero, it runs the `meet` function which should cause
subsequent calls to `is_met` to return zero. Following this pattern makes
Skittle dependencies idempotent. Go ahead and run it again. It still returns
ok.

### Chaining dependencies

Ok, so this example was a bit basic. Creating the directory alone is probably
not enough. Let's ensure that directory is writable only to a user named
'turtle'. We now have to do four things to satisfy `log_dir`:

  1. Ensure the directory exists.
  2. Ensure the turtle user exists.
  3. Ensure the directory is owned by turtle.
  4. Ensure the directory has 755 permissions.

We'll use Skittle's `require` function for this.

``` bash
# ./deps/log_dir.sh

log_dir() {
  dir_path=/var/log/turtle

  require dir_exists
  require dir_ownership
  require dir_permissions

  dir_exists() {
    is_met() {
      ls $dir_path
    }

    meet() {
      sudo mkdir $dir_path
    }
  }

  dir_ownership() {
    turtle_user_exists() {
      is_met() {
        id turtle
      }

      meet() {
        sudo useradd turtle
      }
    }

    is_met() {
      [[ `ls -ld $dir_path | awk '{print $3}'` = "turtle" ]]
    }

    meet() {
      sudo chown -R turtle: $dir_path
    }

    require turtle_user_exists
  }

  dir_permissions() {
    is_met() {
      [[ `ls -ld $dir_path | awk '{print $1}'` = "drwxr-xr-x" ]]
    }

    meet() {
      sudo chmod -R 0755 $dir_path
    }
  }
}
```

Now when we run `skittle log_dir`, we see it executes a tree. Again, this is
idempotent—you can run it over and over just fine.

```
bash-3.2$ ./skittle log_dir
+ log_dir
|
+-+ dir_exists
| |
| \ [ok] dir_exists
|
+-+ dir_ownership
| |
| +-+ turtle_user_exists
| | |
| | \ [ok] turtle_user_exists
| |
| \ [ok] dir_ownership
|
+-+ dir_permissions
| |
| \ [ok] dir_permissions
|
\ [ok] log_dir
bash-3.2$
```

Notice that the original `log_dir` code moved into an inner depedency called
`dir_exists`, since it was only checking if the log directory existed. The
other requirements have then been stated using `require`.

Notice also that `dir_ownership` has a nested dependency `turtle_user_exists`.
This is allowed with Skittle so that you can group related dependencies
together. In this case however, `turtle_user_exists` probably doesn't belong
here, as it is not directly related to the log directory. It is very simple to
just move the function definition to `./deps/turtle_user_exists.sh`. Try it,
everything should work the same.

### Parameterized dependencies

Sometimes it is useful for dependencies to accept arguments, so that they
become more general and re-usable. A great example of this is
`turtle_user_exists` from our earlier example. The code here could work for any
user, so we can generalize it and accept a username as an argument. Because
dependencies are just bash functions, arguments are numbered `$1`, `$2` etc.

``` bash
# ./deps/user_exists.sh

user_exists() {
  username=$1

  is_met() {
    id $username
  }

  meet() {
    sudo useradd $username
  }
}
```

Now we can change `log_dir` to use this generalized dep instead.

``` bash
# ./deps/log_dir.sh

log_dir() {
  dir_path=/var/log/turtle

  # ... snip ...

  dir_ownership() {
    require user_exists turtle

    is_met() {
      [[ `ls -ld $dir_path | awk '{print $3}'` = "turtle" ]]
    }

    meet() {
      sudo chown -R turtle: $dir_path
    }
  }

  # ... snip ...
}
```

> **Note** It is important to store the function arguments to variables so they
>          can be used inside `is_met` and `meet`.

### Logging/debugging

Sometimes you'll want to provide some arbitrary output to the console while
Skittle is processing the dependency tree. Using `echo` won't work, since
Skittle gathers all output on `stdout` and `stderr` into the log file. Instead,
Skittle provides `log`, which prints a message to the console and reflects
which branch of the dependency tree this log output comes from.

You can use `log` anywhere inside the main function body of the dep, or
inside `is_met` and `meet`.

For example, in our generic `user_exists` dep, it might be a good idea to show
which user the dep is being run for.

``` bash
# ./deps/user_exists.sh

user_exists() {
  username=$1

  log "Checking user: $username"

  is_met() {
    id $username
  }

  meet() {
    sudo useradd $username
  }
}
```

This will output something along the lines of.

```
bash-3.2$ ./skittle log_dir
+ log_dir
|
+-+ dir_exists
| |
| \ [ok] dir_exists
|
+-+ dir_ownership
| |
| +-+ user_exists
| | |
| | \ [..] Checking user: turtle
| | |
| | \ [ok] user_exists
| |
| \ [ok] dir_ownership
|
+-+ dir_permissions
| |
| \ [ok] dir_permissions
|
\ [ok] log_dir
bash-3.2$
```

Another suggested use of `log` is to provide feedback to the user during
long running operations.

``` bash
compile_source() {
  cd project_src/

  is_met() {
    [[ -f some_binary ]]
  }

  meet() {
    log "Compiling (may take some time)"
    make
  }
}
```

### Grouping multiple deps together

Ok, so we can create a log directory for our turtle service just fine. You
could go on to install the entire turtle service, by breaking the problem down
into similarly small tasks and putting them all under a top level dependency,
like so.

``` bash
install_turtle() {
  require copy_files
  require runlevels
  require log_dir
  require data_dir
}
```

Running `./skittle install_turtle` will run each of the deps listed in order.
If any single dep fails, the entire install process will halt. It's not hard
to see how something like this could be wrapped under a simple `install.sh` for
users of the turtle service to run.

That's pretty much all there is to it!

### Composing dependency trees

At some point you may want to break your dependencies down into separate
projects in order to allow for re-use in different ways. In general, this is as
simple as adding nested directories under ./deps, however, there is one caveat:
if your deps are coded using relative paths to support files, such as the
following, you may run into trouble nesting your deps, since `$PWD` is relative
to the user running `skittle`, not relative to the dependency file itself.

``` bash
etc_issue() {
  wanted=./deps/issue_file.txt

  is_met() {
    diff $wanted /etc/issue
  }

  meet() {
    sudo cp -f $wanted /etc/issue
  }
}
```

If this dep is nested at, say ./deps/common/deps/etc_issue.sh, it will not do
what you expect due to the relative path.

For this purpose, Skittle provides `$p`, which is implicitly set for each dep
being run. This special variable defines the directory from which the
dependency file itself was loaded. You should use it everywhere you mean
"relative to the current dependency".

``` bash
etc_issue() {
  wanted=$p/issue_file.txt

  is_met() {
    diff $wanted /etc/issue
  }

  meet() {
    sudo cp -f $wanted /etc/issue
  }
}
```

Now the above dep will run anywhere it is copied to.

## Tests

Skittle uses Skittle to test itself. To run the tests, run `skittle` with
the 'tests' dep.

```
./bin/skittle tests
```

Reading through the test code (in the deps directory) is good way to see an
example of Skittle code too.

## Other features

  * Dependencies can be stored in subdirectories
  * Other supported locations for `./deps` are `~/skittle-deps` and
    `./skittle-deps`
  * The dependency function is unset before its `is_met` and `meet` functions
    are run. This makes it safe to name a dependency the same as a system
    binary (i.e. if you execute the system binary `git` in a dep named `git`,
    you will indeed execute /usr/bin/git, rather than recurse into the dep
    function)

## Donations

[![Tip!](http://img.shields.io/gittip/d11wtq.svg)](https://gittip.com/d11wtq)

I work on open source projects for free and because I genuinely enjoy giving to
the community, but of course any donations are well-received. You can donate a
small amount via [gittip](https://gittip.com/d11wtq) if you're feeling generous.

## Copyright &amp; Licensing

Copyright &copy; 2014 Chris Corbyn. See the LICENSE file for details.
