---
title: Finite-state Transducer
date: 2018-12-17 16:55:02
tags: [fst, nlp]
mathjax: true
---

### Overview

> A finite-state transducer (FST) is a finite-state machine with two memory tapes, following the terminology for Turing machines: an input tape and an output tape.

它和一般的finite-state automaton的区别在于“an ordinary finite-state automaton has a single tape”。FST也是一种finite-state automaton，但是它会在两组符号之间完成映射。FST是FSA的一种更为泛化的表达。

FSA通过定义一组可接受的字符串定义了一种形式语言(a formal language)，而FST两组字符串之间的映射关系。

<!-- more -->

一个automaton可以用来识别字符串，也可以用来生成字符串。如果我们将它的tape上的内容作为输入的话，则其可以用来识别字符串（也就是automaton可以将字符串映射在$\{0, 1\}$空间内）。而如果我们将automaton的tape上的内容视为输出的话，它可以被用来生成字符串序列。从这个角度看来，automaton生成了一种形式语言(也就是一组字符串)。

Transducer的两个tape通常被视为一个input tape和一个output tape。从这个角度看，transducer将input tape上的内容转化成了output tape上的内容。它可以具有不确定性，对于一个输入字符串可能产生多个输出字符串。它也可能对给定的输入字符串没有生成任何对应的输出字符串，这种情况称为transducer驳回(reject)了输入。一般来讲，transducer计算了两种形式语言之间的关系。

