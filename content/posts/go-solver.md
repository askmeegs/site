---
layout: post
title: "tiny solver in go"
description:
date: 2018-03-30
tags:
comments: false
---

I think that building small useful things is really gratifying.

The other day, I was working through [a programming problem](https://github.com/m-okeefe/go-ds/blob/master/problems/sheep/main.go) of the [Codejam variety](https://code.google.com/codejam/contest/6254486/dashboard). In the past, I've usually tackled these sorts of problems in Python, but decided that day to work in Go.

Most Codejam problems have the same input and output format -- test file in, first line denotes number of testcases, one line per testcase, then the same for the output file.

And so I decided to write a tiny question solver that handles this common format:

```
// a solver is a function that generates an answer to a programming question
type Solver func(string) string
```

Strangely, despite having spent a few years with Go, I've almost never created a `type` for a function. `Solver` is basically an interface that any programming problem-solver must implement. The `string` input is one test case from the "in-file," and the `string` output is the solved result of that test case.

Then I wrote a function that passes test cases from the input file to a function of type `Solver` --

```
func SolveTestfile(f string, s Solver) error {
	output := []string{}
	file, err := os.Open(f)
	if err != nil {
		return err
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	scanner.Scan() //trash 1st line
	i := 1
	for scanner.Scan() {
		result := s(scanner.Text())
		output = append(output, result)
		i = i + 1
	}
	return writeTestResults(output, fmt.Sprintf("OUT_%s", f))
}
```

And finally just a small unit test that took in this sample "in-file" --
```
3
hello
big
world
```

 -- Tested against an "EchoSolver" that spits back out the input as output, for each test case.

```
package test

import (
	"testing"
)

func EchoSolver(s string) string {
	return s
}

func TestSolve(t *testing.T) {
	err := SolveTestfile("echo_tiny.in", EchoSolver)
	if err != nil {
		t.Error(err)
	} else {
		t.Log("test succeeded")
	}
}
```

The output file:

```
Case #1: hello
Case #2: big
Case #3: world
```

Now when I solve past Codejam problems, I can just import the Solver, and swap in small/large test files easily.  
Yay, tiny useful things!
