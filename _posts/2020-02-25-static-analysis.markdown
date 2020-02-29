---
layout: post
title:  "Basic Static Analysis for bug hunting"
date:   2020-02-25 23:56:00 +0900
categories: clang static analysis bughunting
---

### 어색한 오랜만의 포스팅...

공개적인 곳에 포스팅은 굉장히 오랜만이다. 그래서 도입 부분이 굉장히 어색하다..
오늘 이야기해 볼 것은 `Static Code Analysis`이다.  나의 경우에는 버그헌팅에 대한 방법을 다양하게 알아 보던 중 접하게 되었다. 기존에는 주로 무작위 및 code coverage 기반 fuzzing과 code audit을 통해 취약점을 찾았었는데, Chrome IPC 1day 분석 중 CodeQL이란 `Static Code Analysis` 도구에 대한 정보고 들었고, fuzzer 또한 정적 분석 맞춰 발전하는 것 같아 공부해 보았다. 

**하지만 주관적인 생각으로는 아직까지 clang의 `Static Analyzer`는 bug hunting 용도로는 부족하다. 그 이유에 대해서는 뒤에 언급 한다.**

많은 Static Code Analysis 도구들이 있지만(CodeQL!), 지금은 `clang analyzer`을 알아보고 그리고 실제 `checker`를 만들어 보는걸 목표로 할 것이다.

우선, `clang analyzer`에 대한 간략한 정의는 다음과 같다.

**The Clang Static Analyzer is a source code analysis tool that finds bugs in C, C++, and Objective-C programs.**

위에서 말한 것과 같이 clang analyzer는 C, C++, Obj-C만 지원한다. 다행인 건, 내가 관심 있어 하는 버그 헌팅 대상은 주로 C,C++로 짜여져 있고 open source라는 점이다(!). 다음은 clang에서 지원하는 `static analyzer`에 대한 repository다.

* [https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer](https://github.com/llvm-mirror/clang/tree/master/lib/StaticAnalyzer)

(직관적인 디렉토리 이름에 반가운 마음으로 클릭했다가는 암호 수식과 유사한 c++ 클래스들의 웅장한 모습에 정신이 아찔해질 수도 있다.) 

그럼, 이쯤에서 간략히 clang analyzer를 사용해 보도록 하자. 다음 프로그램은 누구나 알 수 있는 직관적인 버그를 가지고 있다. 바로 Division by zero이다.

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
위 결과에서 볼 수 있듯이 Division by zero 타입의 버그를 바로 찾아내었다. 하지만, 우리가 관심 있어 하는 대상은 보통 프로젝트로 이루어져 있다. 따라서 위처럼 단일 파일 대상으로 clang --analyze 옵션을 사용하는 것은 효율적이지 않다. 이런 경우에는 `scan-build`를 이용해야 한다.

`scan-build`는 clang의 도구로 `기본 설치 되어 있지 않다.`(clang 6.0/ubuntu 18.04 기준으로 default install X) 따라서 `scan-build`를 사용하기 위해 직접 설치해 주어야 한다. 다음 명령으로 설치하면 된다.

```bash
sudo apt install -y clang-tools
```

위 명령이 성공적으로 실행되면, 머신 내에 `scan-build` 프로그램이 설치 되었을 것이다. `scan-build`의 간단한 사용법은 다음과 같다.

```bash
singi@singi-VirtualBox:~/sa$ scan-build -o test2 make
scan-build: Using '/usr/lib/llvm-6.0/bin/clang' for static analysis
/usr/share/clang/scan-build-6.0/bin/../libexec/ccc-analyzer -g -o test test.c
test.c: In function ‘vuln1’:
test.c:6:5: warning: division by zero [-Wdiv-by-zero]
    a/0; //[1]
     ^
test.c: In function ‘main’:
test.c:11:10: warning: implicit declaration of function ‘atoi’ [-Wimplicit-function-declaration]
    vuln1(atoi(argv[1]));
          ^~~~
test.c:6:5: warning: Division by zero
   a/0; //[1]
   ~^~
1 warning generated.
scan-build: 1 bug found.
scan-build: Run 'scan-view /home/singi/sa/test2/2020-02-29-032210-8126-1' to examine bug reports.
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
Makefile 구성이 약간 중요한데, 컴파일러는 **꼭!** $(CC) 형태로 사용해야 한다. 그냥 clang 경로를 적어준다면, scan-build는 어떠한 버그 리포트도 생성하지 않을 것이다.

```plaintext
CC=clang
all:
        $(CC) -g -o test test.c
```

간략히, `scan-build <make command>`를 사용하면 된다. 그런데 여기서 잠깐, 주석 `[1]`의 a/0을 **a/y**로 변경하면 어떻게 될까? 한번 해보자... 취약점이 될 가능성은 있지만 못 잡는다. 이유는 뒷 부분에 설명한다.

### clang static analyzer는 어떤 방법으로 정적 코드 분석을 수행할까?

source code를 컴파일 할 때 AST(`Abstract Syntax Tree`) Node을 추출하고, 이 AST Node의 형태가 미리 정의된(또는 정의 할) `bug type pattern`이 있는지 검색한다. 만약 `bug type pattern`이 존재한다면 이를 `reporting` 한다. 즉, static analyzer는 ***실행 가능한 경로를 trace 하는 소스 코드 시뮬레이터***다. 

우선, source code를 컴파일 할 때 AST가 어떤 식으로 출력 되는지 확인해보도록 하자.  

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

clang static analyzer는 컴파일 시 AST Node를 추출/객체화 한 후, check 과정을 추가 한 것이다. 우리가 할 일은 이 check 과정에서 AST Node를 기반으로 한 `bug type pattern`을 만들어 제공하는 것이다.

이것이 가능한 이유는 clang static analyzer는 symbolic execution을 수행하기 때문이다. 모든 입력값은 symbolic values로 표현되며 analyzer는 input symbol 및 program path에 기반하여 모든 가능한 경로 및 값을 추론한다. 

실행된 경로의 추적은 Graph로 표시 되고, 각 Graph는 `ProgramPoint`, `ProgramState`로 표현된다. 이렇게 표현된 Graph는 새로운 statement가 수행 될 때 마다 추가/삭제/수정 된다.

그럼 우선, checker 중 1개를 분석 해 보도록 하자.

### Division by zero checker 분석

해당 checker의 코드는 다음 git repository에서 확인할 수 있다.
* https://github.com/llvm-mirror/clang/blob/master/lib/StaticAnalyzer/Checkers/DivZeroChecker.cpp

모든 checker는 다음과 같은 코드로 chekcer로 등록될 수 있다.
```c++
void ento::registerDivZeroChecker(CheckerManager &mgr) {
  mgr.registerChecker<DivZeroChecker>();
}
```

이제 `DivZeroChecker` 클래스를 확인 해 보자.

```c++
class DivZeroChecker : public Checker< check::PreStmt<BinaryOperator> > {
  mutable std::unique_ptr<BuiltinBug> BT;
  void reportBug(const char *Msg, ProgramStateRef StateZero, CheckerContext &C,
                 std::unique_ptr<BugReporterVisitor> Visitor = nullptr) const;

public:
  void checkPreStmt(const BinaryOperator *B, CheckerContext &C) const;
};
```

`DivZeroChecker` 클래스에서 `reportBug`, `checkPreStmt` 메소드를 overide 하고 있다. 

`reportBug` 메소드는 `checkPreStmt` 메소드 안에서 bug type pattern이 나오면 user에게 report하는 용도로 사용된다. 따라서 중요한 메소드는 `checkPreStmt` 메소드가 된다. 

`checkPreStmt` 메소드는 check할 AST Node에 따라 다양한 AST Node Class를 arguments로 사용할 수 있다. 

AST Node Class에는 Statement, Exrepssion, Operator등이 포함된다. Division By zero checker의 경우, `BinaryOperator`를 사용했다. 다음은 `checkPreStmt` 메소드다.

```c++
void DivZeroChecker::checkPreStmt(const BinaryOperator *B,
                                  CheckerContext &C) const {
  BinaryOperator::Opcode Op = B->getOpcode();
  if (Op != BO_Div &&
      Op != BO_Rem &&
      Op != BO_DivAssign &&
      Op != BO_RemAssign)
    return;

  if (!B->getRHS()->getType()->isScalarType())
    return;

  SVal Denom = C.getSVal(B->getRHS()); //[1]
  Optional<DefinedSVal> DV = Denom.getAs<DefinedSVal>(); //[2]

  // Divide-by-undefined handled in the generic checking for uses of
  // undefined values.
  if (!DV)
    return;

  // Check for divide by zero.
  ConstraintManager &CM = C.getConstraintManager();
  ProgramStateRef stateNotZero, stateZero; //[3]
  std::tie(stateNotZero, stateZero) = CM.assumeDual(C.getState(), *DV); //[4]

  if (!stateNotZero) { //[5]
    assert(stateZero);
    reportBug("Division by zero", stateZero, C); //[6]
    return;
  }

  bool TaintedD = isTainted(C.getState(), *DV);
  if ((stateNotZero && stateZero && TaintedD)) {
    reportBug("Division by a tainted value, possibly zero", stateZero, C,
              std::make_unique<taint::TaintBugVisitor>(*DV));
    return;
  }

  // If we get here, then the denom should not be zero. We abandon the implicit
  // zero denom case for now.
  C.addTransition(stateNotZero);
}
```

프로그램의 상태는 해당 AST Node의 변수와 표현식으로 구성되고 이것은 `ProgramState`로 나타낼 수 있다. 

위 코드의 주석 `[1]`에서 BinaryOperator의 우변 값(a/0의 경우, 0)을 가져와 SVal(Symbolic Expression)로 변환한다. 그리고 주석 `[2]`에서 지정된 SVal 유형으로 변환한다. 

여기서는 `DefinedSval` Type을 사용한다. 이 `DefinedSval` type은 True/False로 나타낼 수 있는 식을 뜻한다. 

주석 `[3]`에서 `ProgramState`인 `stateNotZero`, `stateZero`를 선언한다. 이 값들은 바로 다음 주석 `[4]`에서 `CM.assumeDual` 메소드의 반환값을 저장하는 용도로 사용된다. 

그럼 주석 `[4]`가 Division by zero bug type pattern 여부를 판단하는 key logic이 된다는 걸 예감할 수 있다. `std::tie` 메소드가 생소할수도 있는데, 이는 C++에서 1개 이상의 메소드 리턴값을 받아올 때 사용한다. (만약, C++17 문법을 지원한다면, 간단하게 auto [x,y]로 표현할 수도 있다.)

`CheckerContext::getState`는 다음과 같이 구현 되어있다.
```c++
const ProgramStateRef &getState() const { return Pred->getState(); }
```

위 메소드는 Graph의 State를 가져오는 간단한 메소드다. 다음으로 `ConstraintManager::assumeDual`은 다음과 같이 구현 되어있다.

```c++
  ProgramStatePair assumeDual(ProgramStateRef State, DefinedSVal Cond) {
    ProgramStateRef StTrue = assume(State, Cond, true);

    // If StTrue is infeasible, asserting the falseness of Cond is unnecessary
    // because the existing constraints already establish this.
    if (!StTrue) {
#ifndef __OPTIMIZE__
      // This check is expensive and should be disabled even in Release+Asserts
      // builds.
      // FIXME: __OPTIMIZE__ is a GNU extension that Clang implements but MSVC
      // does not. Is there a good equivalent there?
      assert(assume(State, Cond, false) && "System is over constrained.");
#endif
      return ProgramStatePair((ProgramStateRef)nullptr, State);
    }

    ProgramStateRef StFalse = assume(State, Cond, false);
    if (!StFalse) {
      // We are careful to return the original state, /not/ StTrue,
      // because we want to avoid having callers generate a new node
      // in the ExplodedGraph.
      return ProgramStatePair(State, (ProgramStateRef)nullptr);
    }

    return ProgramStatePair(StTrue, StFalse);
  }
```

위 메소드는 방금 가져온 그래프 state와 평가해야 할 Condition(DefinedSVal)을 인자로 받는다. 만약 주석 [1],[2]에서 `a/0` statement의 우변 값(0)을 SVal 형식으로 만들었다면, `assume(State, Cond, true);` 메소드 반환값은 False가 된다.

그리고는 주석 `[5]`에서 `stateNotZero`가 False라면 주석 `[6]`의 `reportbug` 메소드를 호출한다.



### 결론?

static analyzer를 통해서 0day 취약점을 찾을 수는 있다. 하지만 많은 배경지식을 요구한다. symbolic execution과 taint analyze는 기본적인 프로그램 분석 기술이라 다양한 곳에 적용 가능 해 배워둘 가치가 충분히 있다.

하지만, clang static analyzer를 사용하기 위해서는 AST Node Class와 clang에서 구현한 symbolic execution 메소드의 사용법을 익혀야 한다.  이를 위해 여러 checker examples이 존재하지만, 너무 복잡하다.

개인 취향의 문제이긴 하지만, 공부용 외에는 clang static analyzer를 실제로 0day bug hunting에 사용하진 않을 것 같다.



### Reference
- [https://chromium.googlesource.com/chromium/src.git/+/master/docs/clang_static_analyzer.md](https://chromium.googlesource.com/chromium/src.git/+/master/docs/clang_static_analyzer.md)
- [https://www.youtube.com/watch?v=UcxF6CVueDM](https://www.youtube.com/watch?v=UcxF6CVueDM)