# Introduction
  As for any compiler / libary pair, only certain versions of the codeql compiler
  will work with certain versions of the libary.  To ensure compatibility, the
  tools creating the db, the query writers (if any), and the query runners should
  all sync up on a particular version.

  In practice, this means installing a consistent set of command-line tools
  across a group/organization.  For query developers, it means using those
  versions with the VS Code plugin.

  This document is a sample walkthrough of such a setup; specifically, it
  illustrates using a codeql bundle to set up both the cli and VS Code.  All the
  steps use unix shell commands, familiarity with the sh-family is assumed.  The
  directory paths chosen here are short and (very likely) unique, so you should be
  able to copy and paste most of the snippets without change.

  There are two other approaches to installing and using the codeql
  command-line tools and associated libraries.  The steps are similar to this
  document, so we just mention them:
  1. cli and library from files
     - download a version of the cli
     - download a /matching/ version of the ql library
     - extract them in parallel directories
     - add the cli directory to the =PATH=
  2. cli from file, library from git clone
     - download a version of the cli
     - in a parallel directory, git clone the ql library
     - check out a /matching/ version of the library (see the list via =git tag -l=)
     - add the cli directory to the =PATH=
  
# CodeQL command-line tool setup via bundle
  Choose one of the cli/library bundles from 
  https://github.com/github/codeql-action/releases, then download and extract it.
  To avoid picking up parallel bundles / ql libraries, add one level of directory
  nesting.  We can also use this extra level to identify the codeql version as
  follows. 
  
```
# We're going to get version 2.7.0, so use that as prefix 
mkdir -p ~/Desktop/demo-cli/codeql270
cd ~/Desktop/demo-cli/codeql270

# Get the bundle
wget https://github.com/github/codeql-action/releases/download/codeql-bundle-20211025/codeql-bundle-osx64.tar.gz
# On mac, also do this:
/usr/bin/xattr -c codeql*.tar.gz
tar zxf codeql-bundle-osx64.tar.gz

# Verify the version
./codeql/codeql --version

# Sanity check -- this should show no output
./codeql/codeql resolve qlpacks |grep -v demo-cli/codeql270
```

# Create database
  Now we can build the ql database.  This requires some source code, we just create a
  simple Python demo app here and use the tools just installed.

  Note that we will need to have Python installed.

## Make a sample Python app
```
cd  ~/Desktop/demo-cli/
cat > hello.py <<EOF

def foo():
    print("Hello World!")
EOF
```

## Set PATH
`export PATH=~/Desktop/demo-cli/codeql270/codeql:"$PATH"`

## Build db via codeql database create
`codeql database create -l python -s . -j 8 -v simple.db`

## Make sure the source is in it
`unzip -v simple.db/src.zip | grep hello`
: should be similar to
37  Defl:N ... .../hello.py

# Run a query against the database
  With the database in place at `~/Desktop/demo-cli/simple.db`, create this
  simple query file

```    
cd  ~/Desktop/demo-cli/
cat > FindFoo.ql <<EOF
/**
* @kind problem
* @id sample/find-foo
*/

import python

from Function f
where f.getName() = "foo"
select f, "the function foo"
EOF
```
and create a [query pack file](https://codeql.github.com/docs/codeql-cli/about-ql-packs/#about-qlpack-yml-files):
```
cd  ~/Desktop/demo-cli/
cat > qlpack.yml <<EOF
name: python-query-test
version: 0.0.1
libraryPathDependencies: codeql-python
EOF
```


  and run it via

```
cd  ~/Desktop/demo-cli/
codeql database analyze                          \
        -v                                       \
        --rerun                                  \
        --format=sarif-latest                    \
        --output python-main.sarif               \
        --                                       \
        simple.db                                \
        FindFoo.ql
```

## Run a query suite against the database
  Available query suites include

* $CODEQL_SUPPORT_LANGUAGE-code-scanning.qls
* $CODEQL_SUPPORT_LANGUAGE-security-extended.qls
* $CODEQL_SUPPORT_LANGUAGE-security-and-quality.qls


As example:
```
cd  ~/Desktop/demo-cli/
codeql database analyze                          \
        -v                                       \
        --rerun                                  \
        --format=sarif-latest                    \
        --output python-simple.sarif               \
        --                                       \
        simple.db                                \
        python-code-scanning.qls
```

# VS Code setup
  Install VS Code following [[https://code.visualstudio.com/docs/setup/setup-overview][the instructions for your platform]].
  
  Install the CodeQL extension; see [[https://code.visualstudio.com/docs/editor/extension-marketplace#_browse-for-extensions][this documentation]] for search instructions.

  [command-line extension handling](https://code.visualstudio.com/docs/editor/extension-marketplace#_command-line-extension-management)

  With the codeql cli and libraries installed and set up, follow these steps to
  ensure the VS Code plugin uses them instead of its defaults.   

  [search-path](https://codeql.github.com/docs/codeql-cli/manual/database-create/#cmdoption-codeql-database-create-search-path)

  Open the sample directory in VS Code
  ```
    cd ~/Desktop/demo-cli/
    open -a /Applications/Visual\ Studio\ Code.app .
  ```

  In VS Code, 
  - Set up the workspace
    : view > command palette > save workspace as > simple.code-workspace

  - Add ql library directory to workspace.  Note the absolute path and adjust for
    your system
    : explorer pane > workspace > add folder to workspace >  `~/Desktop/demo-cli/codeql270/codeql/qlpacks`

  - Add the database to workspace
    : QL tab > add database from folder > `~/Desktop/demo-cli/simple.db`
    or
    : explorer tab > `~/Desktop/demo-cli/` > `simple.db` > right click > codeql: set current database

  - Use the just-installed cli (adjust for your setup)
    : open settings > codeql cli executable > workspace > `~/Desktop/demo-cli/codeql270/codeql/codeql`

  - Run a query
    - open `FindMain.ql`
    - Right click > codeql: run query

* References
  Another short cli overview:
  - codeql cli guides: https://github.com/advanced-security/advanced-security-material/blob/main/code-scanning-guides/setup-codeql-cli.md

  And more advanced topics:
  - external data: https://github.com/hohn/codeql-external-data
  - sandwich tracing: https://github.com/advanced-security/advanced-security-material/blob/main/code-scanning-guides/sandwich-tracing.md
  
