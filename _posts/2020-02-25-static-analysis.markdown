---
layout: post
title:  "Basic Static Analysis for bug hunting"
date:   2020-02-25 23:56:00 +0900
categories: clang static analysis bughunting
---

공개적인 곳에 포스팅은 굉장히 오랜만이다. 그래서 도입 부분이 굉장히 어색하다..
오늘 이야기해 볼 것은 `Static Code Analysis`이다. 나의 경우에는 버그헌팅에 대한 방법을 다양화 하던 중 접하게 되었다.
많은 Static Code Analysis 도구들이 있지만(CodeQL!), 여기서는 경우에는 `clang-analyzer`을 알아보고 장점과 단점 그리고 실제 `checker`를 만들어 보는 것으로 마치려 한다.
