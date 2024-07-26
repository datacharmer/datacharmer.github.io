---
title: A weird testscript behavior.
description: Would `TestMain()` run more than once? It would, if the right combination arises.
date: 2024-07-26
---

I have started writing a [testscript cookbook][ts-cookbook], with the intention of showing most of the interesting uses of cookbook.


The first recipe I engaged with was [Automatic tool build without external commands][recipe1], and while I was explaining (to myself, more than future readers) some of the intricacies of the command line, I recalled a funny behavior that I noticed when I was using `testscript` to test a complex tool that required a beefy initialization procedure, and I wanted to use `TestMain` to achieve it.
The initialization works, but I soon realized that the behavior of the test was sometimes erratical. This happened after I introduced the initialization procedure, while I had no issues when the initialization was created before running `go test`. Thus, I started investigating, and I found out that, since the binary executable for `testscript` operation is the same that was compiled by `go test`, the code in `TestMain()` was being runned more than once. To be specific, it was runned every time a test script was invoking the tool being tested.
At the time of the discovery I was in a hurry, so I just added some code to make sure that the initialization happened only once, and continued with the more important testing.

Now, however, I have some time on hand, and I decided to check what was going on in there.

For this reason, I modified the code of one of a [sample TestMain][wordcount-testmain] to include a silent log of the calls received by the application:


```go

func TestMain(m *testing.M) {
	f, err := os.OpenFile("/tmp/count.txt",
		os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		fmt.Printf("%s\n", err)
		os.Exit(1)
	}
	defer f.Close()
	exitCode := testscript.RunMain(m, map[string]func() int{
		"wordcount": cmd.RunMain,
	})
	_, err = f.WriteString(fmt.Sprintf("%s %v\n", time.Now(), os.Args))
	if err != nil {
		fmt.Printf("%s\n", err)
		os.Exit(1)
	}
	f.Sync()
	os.Exit(exitCode)
}
``` 

And this was the result:

```
$ rm -f /tmp/count.txt

$ go test
PASS
ok  	github.com/datacharmer/wordcount	2.191s

$ cat /tmp/count.txt
2024-07-26 12:14:42.738121 +0200 CEST m=+0.000574288 [wordcount]
2024-07-26 12:14:42.738119 +0200 CEST m=+0.000597547 [wordcount]
2024-07-26 12:14:42.758283 +0200 CEST m=+0.000781849 [wordcount -m -w -l]
2024-07-26 12:14:42.794681 +0200 CEST m=+0.000635278 [wordcount -s]
2024-07-26 12:14:42.81414 +0200 CEST m=+0.000554624 [wordcount]
2024-07-26 12:14:42.839205 +0200 CEST m=+0.000687073 [wordcount -w -l -m]
2024-07-26 12:14:42.864199 +0200 CEST m=+0.000546121 [wordcount]
2024-07-26 12:14:42.880693 +0200 CEST m=+0.000540290 [wordcount -w -l -m]
2024-07-26 12:14:42.906099 +0200 CEST m=+0.000695668 [wordcount -s]
2024-07-26 12:14:42.922528 +0200 CEST m=+0.000522322 [wordcount]
2024-07-26 12:14:42.948458 +0200 CEST m=+0.000538770 [wordcount -w -l -m]
2024-07-26 12:14:42.963985 +0200 CEST m=+0.000551695 [wordcount]
2024-07-26 12:14:42.980613 +0200 CEST m=+0.000566742 [wordcount -w -l -m]
2024-07-26 12:14:42.997354 +0200 CEST m=+0.000558493 [wordcount -l -w -c]
2024-07-26 12:14:43.014073 +0200 CEST m=+0.000537438 [wordcount -w]
2024-07-26 12:14:43.031384 +0200 CEST m=+0.000552502 [wordcount -l]
2024-07-26 12:14:43.04736 +0200 CEST m=+0.000564279 [wordcount -c]
2024-07-26 12:14:43.064259 +0200 CEST m=+0.000668487 [wordcount -m]
2024-07-26 12:14:43.081249 +0200 CEST m=+0.000547902 [wordcount -s]
2024-07-26 12:14:43.097558 +0200 CEST m=+0.000650139 [wordcount -u]
2024-07-26 12:14:43.114071 +0200 CEST m=+0.000540160 [wordcount -o]
2024-07-26 12:14:43.13298 +0200 CEST m=+0.000544590 [wordcount -version]
2024-07-26 12:14:43.13574 +0200 CEST m=+0.000915174 [wordcount -log-file /tmp/go-test-script503752326/script-exists/wordcount.log]
2024-07-26 12:14:44.157539 +0200 CEST m=+1.565199836 [/tmp/go-build3146625816/b001/wordcount.test -test.paniconexit0 -test.timeout=10m0s]
```

What's happening? Every `wordcount` in `count.txt` was invoked in one of the tests within the `./testdata` directory. The options shown are the ones used in the tests.
So we see that `TestMain()` was used more than once during this run: 24 times, and one of those times –the last one– it was entered when the test was named as the default name created by `go test`.



[ts-cookbook]:https://github.com/datacharmer/testscript-cookbook
[recipe1]:https://github.com/datacharmer/testscript-cookbook/blob/main/recipes/basic/run-your-cli-tool-without-pre-build.md
[wordcount-testmain]:https://github.com/datacharmer/wordcount/blob/main/main_test.go#L18,L23
