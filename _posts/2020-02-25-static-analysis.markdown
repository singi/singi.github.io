---
layout: post
title:  "Basic Static Analysis for bug hunting"
date:   2020-02-25 23:56:00 +0900
categories: clang static analysis bughunting
---

공개적인 곳에 포스팅은 굉장히 오랜만이다. 그래서 도입 부분이 굉장히 어색하다..
오늘 이야기해 볼 것은 `Static Code Analysis`이다.  나의 경우에는 버그헌팅에 대한 방법을 다양하게 알아 보던 중 접하게 되었다.

많은 Static Code Analysis 도구들이 있지만(CodeQL!), 지금은 `clang-analyzer`을 알아보고 그리고 실제 `checker`를 만들어 보는 것으로 마치려 한다.

우선, clang-analyzer에 대한 간략한 정의는 다음과 같다.

```
The Clang Static Analyzer is a source code analysis tool that finds bugs in C, C++, and Objective-C programs.
```

위에서 말한 것과 같이 clang-analyzer는 C, C++, Obj-C만 지원한다. 다행인 건, 내가 관심 있어 하는 버그 헌팅 대상은 C,C++로 짜여져 있고, open source라는 점이다. 다음은 clang-analyzer에 대한 repository다.

* https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer

- AST Matcher AST
- clang analyzer 원리
- 
간략하게 clang-analyzer와 기본 checker를 사용해 보도록 하자.

- 간단 프로그램
- 스마트 포인터 예제