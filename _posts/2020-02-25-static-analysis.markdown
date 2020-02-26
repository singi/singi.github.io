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

**The Clang Static Analyzer is a source code analysis tool that finds bugs in C, C++, and Objective-C programs.**

위에서 말한 것과 같이 clang-analyzer는 C, C++, Obj-C만 지원한다. 다행인 건, 내가 관심 있어 하는 버그 헌팅 대상은 C,C++로 짜여져 있고, open source라는 점이다. 다음은 clang-analyzer에 대한 repository다.

* [https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer]

clang-analyzer는 clang의 도구로써 기본 설치 되어 있지 않다. 따라서 사용하기 위해서는 직접 컴파일하거나 바이너리를 다운로드 받아 설치 해야 한다.

## clang-analyzer는 어떤 방법으로 static analyzer를 수행하는 것인가?

간단히 말하자면, source code를 컴파일 할 때 AST syntax을 추출하고, 이 AST syntax를 토대로 미리 정의된(또는 정의할) `bug type pattern`이 있는지 검색한다. 만약 존재한다면 이를 reporting 한다. 우선, source code를 컴파일 할 때 AST가 어떤 식으로 출력 되는지 확인해보도록 하자.  다음은 테스트 프로그램과 AST node를 출력한 결과이다.

```c
//clang -Xclang -ast-dump test.c
#include <stdio.h>

int main()
{
        int a = 10;
        printf("%d\n", a);
        return 0;
}
```

```
`-FunctionDecl 0x565559941d90 <test.c:3:1, line:8:1> line:3:5 main 'int ()'
  `-CompoundStmt 0x565559942058 <line:4:1, line:8:1>
    |-DeclStmt 0x565559941ec8 <line:5:2, col:12>
    | `-VarDecl 0x565559941e48 <col:2, col:10> col:6 used a 'int' cinit
    |   `-IntegerLiteral 0x565559941ea8 <col:10> 'int' 10
    |-CallExpr 0x565559941fa0 <line:6:2, col:18> 'int'
    | |-ImplicitCastExpr 0x565559941f88 <col:2> 'int (*)(const char *, ...)' <FunctionToPointerDecay>
    | | `-DeclRefExpr 0x565559941ee0 <col:2> 'int (const char *, ...)' Function 0x565559933058 'printf' 'int (const char *, ...)'
    | |-ImplicitCastExpr 0x565559941ff0 <col:9> 'const char *' <BitCast>
    | | `-ImplicitCastExpr 0x565559941fd8 <col:9> 'char *' <ArrayToPointerDecay>
    | |   `-StringLiteral 0x565559941f08 <col:9> 'char [4]' lvalue "%d\n"
    | `-ImplicitCastExpr 0x565559942008 <col:17> 'int' <LValueToRValue>
    |   `-DeclRefExpr 0x565559941f38 <col:17> 'int' lvalue Var 0x565559941e48 'a' 'int'
    `-ReturnStmt 0x565559942040 <line:7:2, col:9>
      `-IntegerLiteral 0x565559942020 <col:9> 'int' 0
```

- 
간략하게 clang-analyzer와 기본 checker를 사용해 보도록 하자.

- 간단 프로그램
- 스마트 포인터 예제


## Reference
- [https://chromium.googlesource.com/chromium/src.git/+/master/docs/clang_static_analyzer.md]
- [https://www.youtube.com/watch?v=UcxF6CVueDM]