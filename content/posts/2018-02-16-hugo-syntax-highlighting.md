---
date: 2018-02-16
lastmod: 2018-02-16
title: Testing Hugo Syntax Highlighting
authors: ["davidr"]
categories:
  - hugo
tags:
  - hugo
  - code
  - blog
toc: false
---
Hugo uses [Chroma](https://github.com/alecthomas/chroma) for syntax highlighting, so
as I'm setting up the blog, this post will help make sure the code highlighting looks
correct.

## Python
``` python
def hello_world() -> str:
    return "hello, world"
```

## Haskell
``` haskell
module Main where

main = putStrLn "Hello, World!"
```

## Go
``` go
package main

import "fmt"

func main() {
    fmt.Println("hello world")
}
```

