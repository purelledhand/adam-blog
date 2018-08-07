---
layout: post
title: Pintool을 이용한 CTF 풀이 (withcon 2017 - crackme)
date:   2018-08-06 03:28:20 +0300
description: # Add post description (optional)
img: post-4.jpg # Add image post (optional)
tags: [CTF]
author: # Add name author (optional)
---
## 왜 솔버가 아닌 사이드채널 어택이었는지

IDA로 열어봤을 때, 
1. 의미없는 연산이 굉장히 많았습니다.<br>
2. 솔버를 돌리기엔 correct, fail에 대한 주소값을 찾을 수가 없었습니다.
   <br>correct, fail에 대한 주소가 고정적으로 참조되는 경우에는 솔버로 풀 수 있겠지만, **이 문제의 경우 그 주소가 연산중에 동적으로 결정**되었습니다.



## Pintool 셋팅
참고 : [사이드채널 어택을 위한 핀툴 기본 셋팅]

[사이드채널 어택을 위한 핀툴 기본 셋팅]: https://purelledhand.github.io/pintool/

### **icount예제 컴파일**

핀툴의 예제파일 중 instruction을 카운트해주는 예제인 icount.cpp를 컴파일 환경이 마련되어있는 MyPinTool/MyPinTool.cpp로 복사해줍니다.

    purelledhand@purelledhand:~/tools/pin-3.6/source/tools/SimpleExamples$ cp icount.cpp ../MyPinTool/MyPinTool.cpp

컴파일 후 obj-intel64 디렉토리와 so파일이 생성된 것을 확인할 수 있습니다.

    purelledhand@purelledhand:~/tools/pin-3.6/source/tools/MyPinTool$ make
    purelledhand@purelledhand:~/tools/pin-3.6/source/tools/MyPinTool$ ls obj-intel64/
    MyPinTool.o  MyPinTool.so


### **실행**


    purelledhand@purelledhand:~/tools/pin-3.6$ ./pin -t source/tools/MyPinTool/obj-intel64/MyPinTool.so -- ../../week5/crackme
    PASSCODE : A
    FAILED
    Count 151993

## exploit
```{.python}
from pwn import *
import string
import operator

password = ""
inscount = dict()
FLAG=True
before_cnt = 151993

while FLAG:
    inscount = {}

    for i in range(len(string.printable)):
        p = process(["./pin","-t" ,"source/tools/MyPinTool/obj-intel64/MyPinTool.so", "--", "../../week5/crackme"])
        p.sendline(password + string.printable[i])
        res = p.recv()
        cut = p.recvuntil('Count [')
        if "FAILED" not in cut:
            print "PASSWORD : " + password + string.printable[i]
            FLAG = False
            break;
        cnt = p.recvall()[:-2]
        p.close()

        if cnt > before_cnt :
            inscount[password + string.printable[i]] = cnt
            before_cnt = cnt

    password = sorted(inscount.items(), key=operator.itemgetter(1), reverse=True)[0][0]
    before_cnt = sorted(inscount.items(), key=operator.itemgetter(1), reverse=True)[0][1]
```

## FLAG
    purelledhand@purelledhand:~/tools/pin-3.6$ python ex.py
    PASSWORD : H4PPyW1THC0nCTF!