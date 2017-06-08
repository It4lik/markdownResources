# Bash tricks & good practices
This document is basically a memo for me but can be useful to people who knows the shell scripting basics with ```bash```.
It's a list of some ```bash``` good practices. From the ones that are widely accepted to those who are a bit more unrecognized/unknown.  
A lot of ```bash``` behaviours are default but can be modified (who says it's hard to debug a ```bash``` script ?!). We'll see here some of those.  
We'll focus on ```bash``` but there's actually quite a lot of things listed here that also works with ```zsh``` and ```ksh```.  
Last thing : here we focus on "how to use it ?", not "how is this working ?" (except if it's mandatory for some details...)  

Most of things presented here are extracted from the Linux Documentation Project : http://tldp.org/

## Some thoughts about shell-scripting
- shell-scripting must be used to automate little actions, not developing a whole web graphical web interface
- oftenly, these little actions are actions that we repeat each and everyday
- remember that each line in your bash script will run (at least) one process on your OS. Think about that... (syscalls to kernel, latency, etc.)
- for performance, prefer using ```STDIN```, ```STDOUT```, and ```STDERR``` manipulations rather than reading/writing files
  - Pipes only exists in RAM. Files are on drives. No discussion about performance from that point.
  - Additionally, your process won't need to wait among all other processes of your system to read/write on disk : RAM access is direct.
  - You don't **always** need high performance. **You'd rather prefer readability over performance** in some cases.
- if your script is starting to be too long (something like ~200 lines long ?), ask yourself if you can't use a better tool...
- shell-scripting is low-level and very powerful.
  - that also means "Be careful not to fill your whole RAM with a little mistake."

## Summary

- [Some thoughts about shell-scripting](#some-thoughts-about-shell-scripting)
- [Summary](#summary)
- [Basics](#basics)
- [Purely syntax](#purely-syntax)
- [Some of u-should-know-about-it bash features](#some-of-u-should-know-about-it-bash-features)
  - [Bash types](#bash-types)
  - [Builtins and keywords](#builtins-and-keywords)
  - [Others features](#other-features)
- [General good practices](#general-good-practices)
- [Readability](#readability)
- [Some of bash awesome features](#some-of-bash-awesome-features)
  - [Braces magic](#braces-magic)
  - [Internal variables](#internal-variables)
  - [Boolean line evaluation](#boolean-line-evaluation)
  - [Other powerful features](#other-powerful-features)
- [Details](#details)
- [Tools](#tools)

## Basics
- always use a **shebang**
- always **comment** scripts. Prefer 'functional' comments rather than 'technical'. We don't want to know about each arguments of your ```find``` one-liner.
```shell
$ HTTPD_PIDS=$(ps -ef | grep httpd | cut -d" " -f1)
# Prefer "Retrieve all current Apache processes PIDS in HTTPD_PIDS"
# Rather than "List all current processes running on system, grep then on with httpd in the name, [...] and store the standard output in the HTTPD_PIDS variable."
# (note that this may not be the best way to retrieve Apache processes PIDs...)
```
- use **exit codes** (with ```exit```)
  - in a function **```return``` is NOT for the return value** (which can be returned using ```STDOUT```) **but the exit status of the function**

- run ```bash``` with ```-v``` and ```-x``` on your scripts options for **debugging purposes**. (note this won't affect ```STDIN``` nor ```STDOUT```)
  - ```-v``` will print all lines executed by the script, one by one
  - ```-x``` will go deeper and print each steps executed by ```bash```
  - **of course you can do both !**
  - try ```bash -v -x helloworld``` with the following :

## Purely syntax
- nearly **always use braces with variables** ```${your_var} instead of $your_var ```. **Holy shit, not quotes. NOT QUOTES. Braces.**
  - this isn't actually needed everywhere. But putting it everywhere won't interfer with any existing script and will avoid that shit happens when calling the tenth argument with  ```$10``` for example...
```shell
#!/bin/bash
HELLO="hello"
echo "${HELLO} World !"
```

- **tests**
  - ```test``` and  ```[ ]``` are POSIX-compliant. POSIX is great, POSIX is mighty, but POSIX is not everything. You should probably use...
  - ```[[ ]]``` is the 'modern' shellscript way to issue tests. POSIX doesn't know about it, but this exists for a moment in ```bash```, ```ksh``` and ```zsh```. It also supports way more features (some of these will be discussed later on in this document)
  - **double-quote for substitution** (including variable substitution)  
- There are a few different ways to **declare a function**. Choose one and stick to it.
```shell
func sayHello() { echo "Hello World !" }
func sayHello { echo "Hello World !" }
sayHello() { echo "Hello World !" } # personnally like this one
sayHello{ echo "Hello World !" }
```

## Some of u-should-know-about-it bash features
### Bash types
- Bash know about types. Yeeees he does.
  - by default, all variables are interpreted as strings
  - you can specify a type with ```declare``` builtin's options
```shell
declare -i max          # int. Since this an int, you can execute arithmetic expressions after affectation :
declare -i max=3+4
declare -a temperatures # indexed array
declare -A temperatures # associative array
declare -r doNotTouchMe # read-only variable (re-affectation failed with exit status ```1``` and a message in ```STDERR```)
```

### Builtins and keywords
You must know about them. Start from running ```compgen -b``` and ```compgen -k``` to retrieve a list of all builtins, and a list of all keywords.


### Other features
- ```$1```, ```$2```, ..., ```$n``` contains the args !
  - this works for ```scripts``` but also in functions' bodies
  - ```$@``` contains all args. ```$#``` contains the number of args.
  - Don't forget about ```shift [int]```
- use ```" "``` to enable things like variable and command substitution
- use ```' '``` to make ```bash``` interpret literally the line (except for ```'``` character who is the end of string)
```shell
# stupid example
test="Heeey"
echo '${test}' "is ${test}"
```
- who says there's no **'scope'** concept nor **'variable accessibility'** in ```bash``` ? Okay, that's not much, but shellscripting actually **don't need** more :
  - ```local HELLO="hello"``` limits visibility to the current scope (i.e. the current function)
  - ```readonly HELLO="hello"``` $HELLO is "hello" okay ?!
  - Nearly everything must be local in shell-script
    - Don't hesitate to encapsulate your whole shell script in a ```main()``` function with only local vars
    - The only global command in the code is now ```main```
- ```[[ ]]``` supports a lot of options (like ```-f``` to test if a file exists, or ```-d``` for a directory)
  - it also supports regex using ```=~``` operator. Stupid example :
```shell
[[ $name =~ ^John$]] \
  && echo "Your name is John." \
  || echo "Your name is not John. You've learn something."
```
- **```bash``` can stop his execution if command failed**. Just use ```set -e``` on top of your script.
  - ```set``` supports a few other options, [check them out](http://tldp.org/LDP/abs/html/options.html)
  - it is used to se ```bash``` options from within the script (and eventually unset them)
  - surround a block with ```set -x``` and ```set +x``` to debug only this specific block
- you can **trap kernel signals** with... ```trap```
  - trap ```SIGINT``` (CTRL+C) to cleanup before exiting
```shell
trap cleanupFunction SIGINT

cleanupFunction() { # Do stuff }
```
- There's like million ways to **redirect both ```STDERR``` and ```STDOUT```**. This is the recommend way (by the community, not me :D)
```shell
echo "Wooop" &> logfile # Equivalent to something like 2>&1 > logfile
```
## General good practices
- **Always use a ```usage()``` function** that will be called when using ```-h``` switch.
  - You can also print it when a user calls you script with a non-existing argument  
- Also a good idea to use a ```version()``` function. For the ```--version``` switch
- Never use binaries directly (only keywords, builtins and some very common binaries like ```mv```, ```ls```, ```cp```, etc.)
  - Store their absolute path in a variable after testing their presence :
```shell
[[ $(which git) ]] \
  && gitBin=$(which git) \
  || echo "Git binary not found in \$PATH. Exiting." ; exit 1

$gitBin clone https://github.com/It4lik/bashTricks
```
- Same for files
```shell
[[ -f "/etc/superconf.conf" ]] \
  && configFile="/etc/superconf.conf" \
  || echo -e "Configuration file not found in ${configFile}.\nUse -c to specify a config file." ; exit 1
```
- **Use functions.** Yeaaaah use functions. And over-reuse them. (so think about re-useability when writing them)
- Declare functions on top of your script !
- Add a **header** to your scripts. Author, date, purpose, etc.
- I already said that each command is a process ?...
  - Use ```HELLO="hello"``` not ```HELLO=$(echo hello)``` or some other weird stuff...
  - You don't need ```echo |``` to send to ```STDIN``` of another process
```shell
explicitString="hey,you"
echo "${explicitString} | cut -d',' -f1"
# You should prefer :
cut -d',' -f1 <<< ${explicitString}
```
- **Use ```mktemp``` to create temporary files.**
  - ```mktemp -d``` for a directory
```shell
tempFile=$(mktemp)
# Do your job
# rm ${tempFile} # You don't really need to clean it up, since it will be created in /tmp. Up to you.
```
- Use and abuse of space and line feed escape to make one-liners become multi-line
```shell
echo ${dir} | grep "tmp" | grep -v "debug"
# Prefer :
echo ${dir} \
    | grep "tmp" \
    | grep -v "debug"
  # Note that pipes are at the beginning of each line (NOT end of the previous)
```
- If you absolutely need a ```rm``` or something like that, just over-test the variable passed in argument. You basically don't want to delete your ```/```.
- **Case in ```bash```**
  - Prefer uppercase only for global variables
```shell
readonly ARGS="$@"
```
  - Prefer lowercase and underscores for spaces
```shell
local current_user = $(whoami)
```
## Readability
- A lot of things don't matter for ```bash```, if we compare it to some other programming languages like C. But still :
  - **Declare some of your variables** and comment them
    - Even if it's not mandatory
    - Use [declare](#bash-type) to declare more explicitly your variables
  - **Yeah, that one-liner is cool but...** maybe you'd rather go for readibality (and maybe less performance...)
  - ```WinningNumber=3``` this is an int. ```WinningSentence="I won !"``` this is a string. (```bash``` still don't care about types if you start wondering)
- Passing **arguments to functions** is the same mechanism as args to your script
  - **Rename them**, including in functions. ```$1``` is not a very explicit name ~~isn't it ?~~

## Some of bash awesome features
That's where our brain blow up.

### Braces magic
- ```{}``` supports so many things. Ugly syntax spotted.
  - They are used for variable manipulation/substitution primarily (but also simply "brace expansion")
  - When in brackets, you don't need to call your variables with ```$```.
  - *Example* : default value for your variables :
```shell
sayHello() {
  name=${1:-"unknown user"} # Here we say "take the $1 value if set, otherwise use the default value "unknown user" (':-' operator)
  echo "Hello ${name} !"
}
```
- A few examples to illustrate some substitution tricks
```shell
## String case manipulation
# With ^ for the first letter and ^^ for all letters in string
test="hey you !"
echo ${test^[a-z]} # Will print Hey you !
echo ${test^^[a-z]} # Will print HEY YOU !
# The [a-z] regex-like selector is used to specify the letters you want to change
```
```shell
## Trimming
# With # and %. # trims from left of strings, % trims from right.
# Use a single occurrence to match the shortest match. Double it for the longest
# You'll need to use a regexp expression to match pattern. The syntax is ${var#pattern}
test="zimzimbelimlim"
echo ${test%lim*} # Prints zimzimbelim
echo ${test%%lim*} # Prints zimzimbe
echo ${test#zim} # Prints zimbelimlim
```
```shell
mkdir -p /data/{john,mary} # Creates both directories
```
- Other examples : (thx [@ShoxX](https://github.com/shoxxdj))
```shell
$ echo {a,b}{c,d}
ac ad bc bd
```
```shell
$ echo a{b,c}e
abe ace
```
```shell
$ echo {a..z}
a b c d e f g h i j k l m n o p q r s t u v w x y z
```
```shell
$ echo {0..100..5}
0 5 10 15 20 25 30 35 40 45 50 55 60 65 70 75 80 85 90 95 100

```
```shell
# This one doesn't work with zsh
$ echo {a..z..2}
a c e g i k m o q s u w y
```
- But also :
```shell
echo {01..10}
01 02 03 04 05 06 07 08 09 10
# By the way, once more, you don't need echo to iterate over this list :
for numbers in {01..10}; do [...]
# This is brace expansion, interpreted before command execution in a line
# That means that {01..10} will actually be replace with his real value before line execution
```

- Don't forget that ```bash``` understands some metacharacters to match files
```shell
ls {doc,log}? # Will match any file that start with 'doc' or 'log' and followed by any character (only one)
```


### Internal variables
[TDLP reference](http://tldp.org/LDP/abs/html/internalvariables.html)
- demonstration of ```PIPESTATUS``` : get the exit codes of piped commands
```shell
#!/bin/bash
## Purpose : I'm gonna backup that database !
#
# Stuff
#
$MYSQLDUMP -u $USER -h $HOST -p$PASS $myDatabase | $GZIP -9 > $BACKUP_FILE
[[ ${PIPESTATUS[0]}]] || echo "mysqldump command failed. Exiting." ; exit 1
```

### Boolean line evaluation
- ```bash``` is a guy who loves boolean logic
  - he is playing at evaluating every line as a boolean expression
  - a command is evaluated as ```true``` if the exit code is ```0```. If different from ```0```, it's ```false```
  - you can play with ```&&``` and ```||``` boolean operator to make cool things
  - when ```bash``` sees a line, he's trying to answer the question "mmmh, is this **whole line** is ```true``` or ```false``` ?" that really is its only purpose. Maybe it will execute your commands to know :)
    - ```bash``` reads from left to right
    - he first determines how he's gonna make the calculation. He gives a sense to the line by using things like operator priority or parenthesis
    - **```bash``` only executes the commands that are needed to determine if the whole line is ```true``` or ```false```**
  - *Eeeeeexample* (well there's actually already other examples on that page. You probably noticed them if you're not familiar with this notion)
```shell
1   [[ $(which git) ]] \
2      && gitBin=$(which git) \
3      || echo "Git binary not found in \$PATH. Exiting." ; exit 1
```

- This only works because everything we said before. Let's dive in :  
  - ```bash``` evaluates that if [[ $(which git) ]] is true (that means that it found git)
    - he just needs to check the ```&&``` part which is the only which is meaningful from now
    - because nor ```|| false``` nor ```|| true``` can make the **whole line** ```false```
    - if line 1 succeeds, line 2 will be executed, line 3 won't
  - but if it's ```false```, if it did not found ```git```
    - he only needs to check the ```||``` part. Because it's the only part who can still make the **whole line** ```true```
    - if line 1 fails, line 2 won't be executed, but line 3 will
  - hey ! That's an ```if/else``` block !

### Other powerful features
- ```read``` is awesome. Just check the whole ```man```. Classic use :
```shell
while read lines {
  echo ${lines}
} <<< /your/file
```

## Details...
... but maybe not that small
- You can get the name of the current function in ```$FUNCNAME```. Can be pretty handy for more error verbosity.
- ```time``` is a builtin
  - but you also probably got a command. Go try it : ```$(which time)``` it give muuuuch moooore information.
- You can count the number of values in an array with ```#``` key :
```shell
testArray=(1 8 9)
echo ${testArray[*]} # Prints 1 8 9. ${testArray[@]} also works
echo ${#testArray} # Prints 3
```
- ```compgen``` and ```complete``` are the major utilies to manage auto-completion
  - one more you-must-known-about-it feature : listing autocompletion with no arguments, using ```compgen``` will let you lists all entities of specific type
```shell
compgen -k # list keywords. Shortcut for compgen -A keywords
compgen -b # list builtins. Shortcut for compgen -A builtins
# -e is exported variables, shortcut for compgen -A export
# and the list goes on (alias, arrayvar, hostname, job, directory, enabled, running, user, etc.)
```
- Parse options with ```getopts``` (builtin), not ```getopt```
  - To treat multi-character options, just write a loop that converts multi-character to single-character option

## Tools
Some of the few awesome tools for better shell-scripts. These have only been tested with ```bash``` (but works for some others, just refer to the tools' official documentation)
- [```shfmt```](https://github.com/mvdan/sh) for shell format
  - Suggests you a better indentation
- [```shunit2```](https://github.com/kward/shunit2) for unit tests
- [```shellcheck```](https://www.shellcheck.net/) for good practices analyzing and linting (this is purely awesome)
