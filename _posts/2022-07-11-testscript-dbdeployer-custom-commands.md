---
title: Enhancing testscript tests with custom commands and conditions
description: Testing dbdeployer using testscript custom commands and conditions
date: 2022-07-11
---

In the [previous][attempt1] [posts][attempt2] we have used built-in `testscript` commands and conditions.

Some commands are `exec`, `stdout`, `exists`. These commands are convenient, but they can't do everything we need for our
testing. Fortunately, `testscript` allows users to create their own commands. There are two ways of adding custom commands:
* Using the `commands` parameter in [RunMain][runmain] within a `TestMain` function, which allows to define commands that take no arguments and return an integer
* Using the `Cmds` field in `testscript.Params`, allowing the creation of commands that can have several arguments.

It's important to consider that `testscript` commands are **assertions**: a regular execution will have no consequence,
but failing the assertion will terminate the test.

For example: `exec OS-command` will run the command and continue the test if "OS-command" exists and its execution returns
a zero exit code. If "OS-command" does not exist or its execution ends with a non-zero code, the test stops.

We have only seen one condition: `[exec:program_name]`:

```
[!exec:dbdeployer] skip 'dbdeployer executable not found'
```
A condition is a command that returns `true` or `false`. The command used after it is only executed if the condition is
true. The built-in conditions don't offer much useful material for our needs, and thus we create a few of our own, also
using a field in `testscript.Params`.

Let's start by defining a command:

```go
// findErrorsInLogFile is a testscript command that finds ERROR strings inside a sandbox data directory
func findErrorsInLogFile(ts *testscript.TestScript, neg bool, args []string) {
	if len(args) < 1 {
		ts.Fatalf("no sandbox path provided")
	}
	sbDir := args[0]
	dataDir := path.Join(sbDir, "data")
	logFile := path.Join(dataDir, "msandbox.err")
	if !dirExists(dataDir) {
		ts.Fatalf("sandbox data dir %s not found", dataDir)
	}
	if !fileExists(logFile) {
		ts.Fatalf("file %s not found", logFile)
	}

	contents, err := ioutil.ReadFile(logFile)
	if err != nil {
		ts.Fatalf("%s", err)
	}
	hasError := strings.Contains(string(contents), "ERROR")
	if neg && hasError {
		ts.Fatalf("ERRORs found in %s\n", logFile)
	}
	if !neg && !hasError {
		ts.Fatalf("ERRORs not found in %s\n", logFile)
	}
}
```

The function `findErrorsInLogFile` examines the database log in a sandbox deployed by dbdeployer, and returns non-error
when the log contains at least one occurrence of the word "ERROR". In order to use such command, we need to give it a name
and to inform `testscript` of its existence.

```go
func TestAttempt3(t *testing.T) {
	testscript.Run(t, testscript.Params{
		Dir: "testdata",
		Cmds: map[string]func(ts *testscript.TestScript, neg bool, args []string){
            // find_errors will check that the error log in a sandbox contains the string ERROR
            // invoke as "find_errors /path/to/sandbox"
            // The command can be negated, i.e. it will succeed if the log does not contain the string ERROR
            // "! find_errors /path/to/sandbox"
            "find_errors": findErrorsInLogFile,
	    })
    }
}
```

In a similar way, we can create commands that perform several checks:
* Make sure that the sandbox is using as many ports as expected;
* Check that a give list of files in a directory exist;
* Sleep unconditionally a given number of seconds

You can see the full implementation on [GitHub][attempt3].

In a similar manner we can define conditions. It is another field in `testscript.Params`, and requires a function defined as
`func(condition string) (bool, error)`. The documentation is not forthcoming about what to put in such function, and by
a process of trial-and-error I was able to come out with a working example:

```go
func customConditions(condition string) (bool, error) {
	elements := strings.Split(condition, ":")
	if len(elements) == 0 {
		return false, fmt.Errorf("no condition found")
	}
	name := elements[0]
	switch name {
	case "minimum_version_for_group":
		if len(elements) < 2 {
			return false, fmt.Errorf("condition 'minimum_version_for_group' requires a version")
		}
		version := elements[1]
		if strings.HasPrefix(version, "5.7") || strings.HasPrefix(version, "8.0") {
			return true, nil
		}
		return false, nil

	case "exists_within_seconds":
		if len(elements) < 3 {
			return false, fmt.Errorf("condition 'exists_within_seconds' requires a file name and the number of seconds")
		}
		fileName := elements[1]
		delay, err := strconv.Atoi(elements[2])
		if err != nil {
			return false, err
		}
		if delay == 0 {
		    return fileExists(fileName), nil	
        }
		elapsed := 0
		for elapsed <= delay {
			time.Sleep(time.Second)
			if fileExists(fileName) {
				return true, nil
			}
			elapsed++
		}
		return false, nil

	default:
		return false, fmt.Errorf("unrecognized condition name '%s'", name)

	}
}
```

In this function we define two conditions:
* `minimum_version_for_group`, with the version passed as argument, returning true if the given version supports group replication.
* `exists_within_seconds`, with two arguments: a file name and a delay in seconds, returning true if the file exists before such delay expires.

The function accepts one string as argument, and we can treat that string however we please to get the condition name and 
its parameters, if any. In this case, I decided to follow the example of the built-in "`[exec:filename]`", where the
parameter is separated from the name by a colon (":").

These conditions can be used as shown in the sample group replication template:

![](<../images/listing2.jpg>)
![](<../images/listing3.jpg>)
(See code at <https://github.com/datacharmer/testscript-explore/blob/main/attempt3/templates/group.tmpl>)

Before the action starts, we check that the version is among the ones that support group replication. In the resulting
testdata file, the condition would be this:

```
[!minimum_version_for_group:5.6.41] skip 'minimum version for group replication not met'
```
The test will be skipped because group replication requires 5.7. (Note: there is a more precise way of checking the
version eligibility for this feature, but for now this is enough).

Finally, this is the full code for the testing function:

```go
func TestAttempt3(t *testing.T) {
	if dryRun {
		t.Skip("Dry Run")
	}
	// Directories in testdata are created by the setup code in TestMain
	dirs, err := filepath.Glob("testdata/*")
	if err != nil {
		t.Skip("no directories found in testdata")
	}
	for _, dir := range dirs {
		t.Run(path.Base(dir), func(t *testing.T) {
			testscript.Run(t, testscript.Params{
				Dir:       dir,
				Cmds:      customCommands(),
				Condition: customConditions,
			})
		})
	}
}
```

### Summing up

We have implemented several powerful additions for our tests.
However, we still haven't addressed the problem of initializing the environment with the database versions that we want
to use in the tests. I am not sure if it can be solved, but I will try. 

 [attempt1]: https://datacharmer.github.io/testscript-dbdeployer-first-attempt/
 [attempt2]: https://datacharmer.github.io/testscript-dbdeployer-with-templates/
 [runmain]: https://pkg.go.dev/github.com/rogpeppe/go-internal@v1.8.1/testscript#RunMain
 [attempt3]: https://github.com/datacharmer/testscript-explore/blob/main/attempt3/attempt3_test.go
