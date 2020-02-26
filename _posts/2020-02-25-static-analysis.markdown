---
layout: post
title:  "Basic Static Analysis for bug hunting"
date:   2020-02-25 23:56:00 +0900
categories: clang static analysis bughunting
---

## 어색한 오랜만의 포스팅...

공개적인 곳에 포스팅은 굉장히 오랜만이다. 그래서 도입 부분이 굉장히 어색하다..
오늘 이야기해 볼 것은 `Static Code Analysis`이다.  나의 경우에는 버그헌팅에 대한 방법을 다양하게 알아 보던 중 접하게 되었다. 기존에는 주로 무작위 및 code coverage 기반 fuzzing과 code audit을 통해 취약점을 찾았었는데, Chrome IPC 1day 분석 중 CodeQL이란 `Static Code Analysis` 도구에 대한 정보고 들었고, fuzzer 또한 정적 분석 맞춰 발전하는 것 같아 공부해 보았다.

많은 Static Code Analysis 도구들이 있지만(CodeQL!), 지금은 `clang analyzer`을 알아보고 그리고 실제 `checker`를 만들어 보는 것으로 마치려 한다.

우선, `clang analyzer`에 대한 간략한 정의는 다음과 같다.

**The Clang Static Analyzer is a source code analysis tool that finds bugs in C, C++, and Objective-C programs.**

위에서 말한 것과 같이 clang-analyzer는 C, C++, Obj-C만 지원한다. 다행인 건, 내가 관심 있어 하는 버그 헌팅 대상은 주로 C,C++로 짜여져 있고 open source라는 점이다(!). 다음은 clang에서 지원하는 `static analyzer`에 대한 repository다.

* [https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer](https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer)

(직관적인 디렉토리 이름에 반가운 마음으로 클릭했다가는 암호 수식과 유사한 c++ 클래스들의 웅장한 모습에 정신이 아찔해질 수도 있다.) 

그럼, 이쯤에서 간략히 clang analyzer를 사용해 보도록 하자. 다음 프로그램은 누구나 알 수 있는 직관적인 버그를 가지고 있다. 바로 Divided by zero이다.

```c
#include <stdio.h>

int main()
{
   int a = 10;
   a/0;
}
```
clang analyzer는 다음과 같이 사용할 수 있다.
```plaintext
singi@singi-VirtualBox:~$ clang --analyze test.c
test.c:6:3: warning: Division by zero
        a/0;
        ~^~
1 warning generated.
singi@singi-VirtualBox:~$
```
위 결과에서 볼 수 있듯이 division by zero 타입의 버그를 바로 찾아내었다. 하지만, 우리가 관심 있어 하는 대상은 보통 프로젝타 단위의 source code들이기 때문에 위처럼 1개 파일 대상 clang --analyze 옵션을 사용하는 것은 효율적이지 않다. 이런 경우에는 `scan-build`를 이용해야 한다.

`scan-build`는 clang의 도구로 `기본 설치 되어 있지 않다.`(clang 6.0/ubuntu 18.04 기준으로 default install X) 따라서 `scan-build`를 사용하기 위해서 직접 설치해 주어야 한다. 다음 명령으로 설치하면 된다.

```bash
sudo apt install -y clang-tools
```

위 명령이 성공적으로 실행되면, 머신 내에 `scan-build` 프로그램이 설치 되었을 것이다. `scan-build`의 간단한 사용법은 다음과 같다.

```bash
singi@singi-VirtualBox:~/sa$ scan-build make
scan-build: Using '/usr/lib/llvm-6.0/bin/clang' for static analysis
gcc -o test test.c
test.c: In function ‘vuln1’:
test.c:6:3: warning: division by zero [-Wdiv-by-zero]
  a/0;
   ^
test.c: In function ‘main’:
test.c:11:8: warning: implicit declaration of function ‘atoi’ [-Wimplicit-function-declaration]
  vuln1(atoi(argv[1]));
        ^~~~
scan-build: Removing directory '/tmp/scan-build-2020-02-26-044559-23003-1' because it contains no reports.
scan-build: No bugs found.
singi@singi-VirtualBox:~/sa$
```

```c++
//test.c
#include <stdio.h>

void vuln1(int y)
{
   int a = 10;
   a/0; //[1]
}

int main(int argc, char *argv[])
{
   vuln1(atoi(argv[1]));
}
```
간략히, `scan-build <make command>`를 사용하면 된다. 그런데 여기서 잠깐, 주석 `[1]`의 a/0을 **a/y**로 변경하면 어떻게 될까? (...버그는 있지만 못 잡는다. 이유는 뒷 부분에 설명한다.)

## clang analyzer는 어떤 방법으로 static analyzer를 수행하는 것인가?

간단히 말하면, source code를 컴파일 할 때 AST(`Abstract Syntax Tree`) Node을 추출하고, 이 AST Node의 형태가 미리 정의된(또는 정의 할) `bug type pattern`이 있는지 검색한다. 만약 존재한다면 이를 `reporting` 한다. 우선, source code를 컴파일 할 때 AST가 어떤 식으로 출력 되는지 확인해보도록 하자.  

다음은 테스트 프로그램과 AST node를 출력한 결과이다.

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
다음은 AST Node들이다. 출력 되는 양이 너무 많아 main 함수에 관련된 것만 추려내었다.
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

위 결과에서 `FunctionDecl`, `CompoundStmt`, `DeclStmt` 등을 AST Node라 한다. 그리고 뒤에 부가적인 정보가 따라온다. <데이터 타입, 문자열, 함수 이름, ...>

clang-analyzer는 컴파일 시, 위 AST Node를 객체화 한 후, ~~~~
- 스마트 포인터 예제


## Reference
- [https://chromium.googlesource.com/chromium/src.git/+/master/docs/clang_static_analyzer.md](https://chromium.googlesource.com/chromium/src.git/+/master/docs/clang_static_analyzer.md)
- [https://www.youtube.com/watch?v=UcxF6CVueDM](https://www.youtube.com/watch?v=UcxF6CVueDM)