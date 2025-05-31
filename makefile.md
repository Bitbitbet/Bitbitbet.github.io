# Makefile
This document is a brief introduction to Makefile syntax .


## Basic syntax
### Targets
```Makefile
target: [dependency list]
	commands
```
* target: A target. Its commands will be executed with `make target`.
* Dependency list: A space-separated list of files needed by the target. If the file does not exist, the corresponding target, which is the same name as the file, will be executed. The process is recursive.
* If one of the files in dependency list does not exist or more recent than the corresponding file for the current target, the current target will be executed. Otherwise it will not be executed. The rule applies to the execution both direct use of `make target` and due to dependencies.

### Phony targets
syntax:
```
.PHONY: target
target: [dependency list]
	commands...
```
The `.PHONY: target` are used when the `target` never corresponds a file. Even though the dependencies are satisfied and the corresponding file exists, phony targets will not be discarded. Phony targets are usually regarded as functions or the *sub commands* of `make`.

### Variables
syntax: 
```Makefile
# Setting (defining)
# variable = ...
# variable := ...
# variable ?= ...

# Using (expandation)
variable = hello, world
.PHONY: test
test:
	@echo $(variable) # equivalent as ${variable}; @ means do not to echo the command
```
Here, `make test` outputs: `hello, world`.

#### Differences between `=` `:=` and `?=`:

```Makefile
a = b
str1 = ${a}bcdefg
str2 := ${a}bcdefg
a = a

.PHONY: test
test:
	@echo ${str1}
	@echo ${str2}
```
The output will be:
```
abcdefg
bbcdefg
```
That's because, when there's a variable expandation in the content of variable:
* `str1 = ...` is called *lazy expansion*. Value of `str1` will change when the value of `a` changes.
* `str2 := ...` is called *immediate expansion*. The value of `str2` is stable and will not change as `a` changes.
* `a ?= ...` has the same effect as `=` when the variable `a` is not defined. Otherwise it does nothing.



Note that the names introduced by targets are globally visible, ~~while those introduced by variables are probably not~~. When a variable is defined, ~~only the part of the Makefile below the variable definition can the variable be used in.~~

`Whatever, it is smart to put variable definitions before where they are using.`

---
An example:
```Makefile
BINARY=program

.PHONY: build
build: $(BINARY)

.PHONY: clean
clean:
	rm *.o $(BINARY)

function.o: function.cpp function.h
	g++ function.cpp -o function.o

main.o: main.cpp function.h
	g++ main.cpp -o main.o

$(BINARY): main.o function.o
	g++ main.o function.o -o $(BINARY)
```

When `make build` is executed, `make` will check the target `program`, `main.o` `function.o`, `main.cpp` etc. . The process is recursive and all targets and files involved will be checked.

~~The definition of variable `BINARY` must be placed before the line where it is used for the first time. Otherwise unexpected errors will occur, because references of undefined variables are treated as blank.~~

Here are two phony targets: `build` and `clean`. When `make build`(`make` is also ok since `build` is the first target in the file) is executed in a shell, the build of the project will begin. When `make clean` is executed, `make` will clean the built binaries.

## Advance
### Multiple Targets
Basic syntax:
```Makefile
target1 target2: dep # space separated
	cmd
```
Same as:
```Makefile
target1: dep
	cmd
target2: dep
	cmd
```
This means that `target1` and `target2` has the same rule.
### Automatic variables
- `$@`: the target name
- `$?`: files in the dependency list more recent than the target
- `$^`: all files in the dependency list, with duplicates removed
- `$+`: all files in the dependency list, with duplicates kept
- `$<`: The first file in the dependency list

Note that when using `$@` with multiple targets, `$@` will not select multiple targets. Only one of the targets will be selected.

### Wildcard
There are 2 wildcards in Makefile: `*` and `@`.

---
- `*` will match files in the filesystem. Consider the following example.
```Makefile
SRCS := *.cpp
SRCS2 := $(wildcard *.cpp) # put it in the wildcard function

.PHONY: test
test:
	echo $(SRCS)
	echo $(SRCS2)
```
When there are two files with suffix `.cpp`: `a.cpp`, `b.cpp`, the output will be
```
a.cpp b.cpp
a.cpp b.cpp
```
When there is not any files with suffix `.cpp`, the output will be
```
*.cpp			# the pattern is printed
			# yeah, empty
```
It is generally recommended to put wildcards in the `wildcard` function.

---
- `%` is a little bit complex. Consider the following example.
```Makefile
OBJS = a.o b.o c.o

.PHONY: build
build: $(OBJS)
	@echo generate executable
	gcc $(OBJS)

$(OBJS): %.o: %.c
	@echo compile $< to $@
	gcc $< -o $@ -c

a.c:
	echo 'int main() {return 0;}' > $@

%.c:
	touch $@
```
