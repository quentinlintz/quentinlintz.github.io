---
title: Lessons Learned Maintaining an Open Source Project 
date: 2024-08-24 22:00:00 -0500
image:
  path: https://utfs.io/f/ae4ea563-bdeb-4485-b971-89ae09df6f4f-svamd8.png
categories: ["development"]
tags: ["golang", "github", "developer experience"]
---

Working exclusively in the terminal has always been a flex. I think it demonstrates understanding of how linux works if you're using `ps`, or understanding networking if you're using `netstat`, or understanding filesystem search with `grep` and some esoteric pattern you totally _did not_ ask ChatGPT to generate for you. I've also been "bitten by the bug" over the past few months to the point where I'm writing this post in Neovim with no spelcheck[^0]. In fact, my entire motivation for learning Go was to create a command line tool no matter how trivial.

## A Bit about Go

So you don't have to look it up, Go (Golang) is a programming language developed by Google in 2009. It has clean and minimal syntax and offers a "batteries included" experience. The standard library is incredible. If you ever need to install an external library, you can install from GitHub which is used as the Go library repository.

```bash
go install github.com/username/repository
```

Speaking of the tooling, Go has a simple interface for running tests `go test` and managing deps `go mod`. Go is statically typed, which provides the structure and organization I love in programming. The performance is supposedly comparable to C/C++. The LinkedIn Learning course taught me a little concurrency with goroutines, but I didn't use that for this open source project.

Probably my favorite aspect of Go is the ability to build binaries for any operating system or architecture easily.

## cmdtop (Command Top)

Like I said, I learned the basics of Go on LinkedIn Learning. It felt more like getting acquainted with the syntax than trying to understand a new, complex concept. Some of the basic things that I could imagine any low(er)-level language could be used for is parsing files. In fact, parsing is a pretty common coding challenge. I wanted to create something _I_ would use, too. If I could run a command, parse some kind of system metric, and print it so it could be shown off, I would consider the effort a success.

What I was partially inspired by is a command line tool used to flex your OS config on Reddit called [fastfetch](https://github.com/fastfetch-cli/fastfetch) check it out for yourself and see my current settings below.

<img src="https://utfs.io/f/7cb0fdcf-a810-4e46-84dc-89b6c7a10d0f-cen1e.png" alt="fastfetch output" />

Anyway, back to cmdtop. Imagine printing what commands you've been using (like I just told you about my affinity for fastfetch 😏) to share with others who might be interested in learning about your workflow. Or, maybe your tools say something about your software/job niche. A hardware designer might be using the `iverilog` tool or some open source wave viewer like `gtkwave` and could show they call those often.

`cmdtop` will print a list of your most used commands from your shell history. It's built using _only_ Go's standard library. There are ~3 main shells that store command history differently, so this was a challenge with decent scope but still achievable. Oh, star this on GitHub before you forget...

<center><a href="https://github.com/quentinlintz/cmdtop"><img src="https://gh-card.dev/repos/quentinlintz/cmdtop.svg" alt="cmdtop GitHub repo card"></a></center>

Here's an example I just ran:

```
Top 5 commands:
1: yay (50)
2: cd (29)
3: ls (27)
4: vi (24)
5: fastfetch (17)
```

_Riveting, right?_

How many projects aren't started because they initially seem modest? My intention is to compel other developers who are also learning Go to contribute and learn with me. With this goal in mind, I set this project up differently that I normally would -- I'll explain in the next section 😉

The three main shells that I sought to support are: bash, zsh, and fish. I've only used bash and zsh but they store command history differently. This "modest" project would need:

1. **Config**. Or, at least pulling an environment variable to know what shell the user is using
2. **Abstraction**. A parser performs the same function, but its implementation varies based on file format
3. **Data Model**. The output of the parser must be stored in some standard way in order to be printed to the console
4. **Error Handling**. What if the user is using a parser that's not supported, yet? What if we just fail to read one of their environment variables?
5. **Delivery**. How would I run this project if I didn't have the source on my machine to begin with?

You must now be convinced there's always more under the surface of a plum software idea.

I wanted to make an initial version of this file, but as it _may be_ useful to others I wanted to dress it up to be inviting to work on. For me, the more difficult aspect of building `cmdtop` wasn't the software design, but convincing potential contributors it was worth their time contributing. How would I manage that?

## What makes a project inviting?

If you've worked on a software project before, I'm certain you've noticed what makes a project enjoyable to work on. It's easy to write off these points as _minutiae_, but I think it makes projects stand out and is conducive to community growth[^1].

### Documentation and Information

A README is like a project's homepage on GitHub (literally). It makes a first impression on users who visit the repo. I ripped a template and modified it to a few sections I thought were most important: About, Getting Started, Usage, Roadmap, Contributing, License, and Contact.

Not only do users need to know how to get set up to run the code themselves, they need to know it's _actively_ being worked on an there's a current plan of features that should be implemented. I implemented the zsh parser, but wanted to open it up for contributors to implement the others. I added empty checkboxes for those items.

I find it interesting that sometimes contributors may forget to update the documentation of projects despite implementing a new feature. It's almost like it's easy to slip one's mind. It deserves to be just as important, though.

### Patterns and Automation

Contributors shouldn't have to care about other aspects of a project when they're focusing on implementing a feature of fixing a bug, it should just _work for them_. What I really mean is work should be isolated in a way that lets someone flow. 

This is where abstraction shines. A Parser can be generalized to an interface and this is literally it:

```go
package history

import (
	"fmt"
	"log"
	"sort"

	"github.com/quentinlintz/cmdtop/config"
	"github.com/quentinlintz/cmdtop/models"
)

// Parsers will implement ParseHistory which should result in identical
// output regardless of shell
type Parser interface {
	ParseHistory(filePath string) ([]models.Command, error)
}

func PrintTopCommands(cfg config.Config, p Parser) {
	commands, err := p.ParseHistory(cfg.HistoryPath)
	if err != nil {
		log.Fatalf("Error parsing history: %v", err)
	}

	sort.Slice(commands, func(i, j int) bool {
		return commands[i].Count > commands[j].Count
	})

	fmt.Printf("Top %d commands:\n", cfg.Top)
	for i, cmd := range commands {
		if i >= cfg.Top {
			break
		}
		fmt.Printf("%d: %s (%d)\n", i+1, cmd.Name, cmd.Count)
	}
}
```

Contributors know what their implementation should already begin to look like given this defines a method `ParseHistory` with a particular method signature. They can see this single file and begin considering what their feature should look like.

I'm particularly proud of this automated test I wrote that uses a slice of structs for testing each parser implementation:

```go
func TestParseHistory(t *testing.T) {
	tests := []struct {
		name        string
		parser      Parser
		historyFile string
	}{
		{"Zsh", &ZshParser{}, "zsh_history"},
		{"Bash", &BashParser{}, "bash_history"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			testFile := filepath.Join("testdata", tt.historyFile)
			testParser(t, tt.parser, testFile, expectedCommands)
		})
	}
}
```

The outputs of each parser should look the same given the test data is set up consistently, so when bash was implemented, the contributor only had to add one line `{"Bash", &BashParser{}, "bash_history"}` to validate it worked correctly. The test pattern is already fleshed out.

### Care and Attention

It might seem obvious that if you _build_ something you're giving it your attention. But when you consider yourself done, what then? When you're interested in encouraging contributors, the time you spend responding matters greatly. Contributors deserve a quicker turnaround time on a PR (in my opinion). Reviewing it as soon as you can shows you value their work. Even if you suggest changes, I think it's likely to be completed because they know someone cares about their effort. Be sincere.

Issues will be created, whether by you or someone else. Those can be categorized sometimes in a helpful way with `good first issue` tags. That can really demonstrate attention to planned work. There are helpful bug report templates that I see used in popular open source projects. I haven't really graduated to that level, yet, but I'm always impressed by how helpful they are for developers who want to try tackling a bug.

Considering what makes a project interesting and inviting to other developers has really been on my mind lately, hope this was helpful in some way and you can include these principles in your own project. I believe they extend even beyond open source projects. Also, please share your `cmdtop` output with me 😁

## Footnotes

[^0]: Just learned about `:set spell` in Vim 😂
[^1]: My project has only 2 other contributors, I might just be scratching the surface here

