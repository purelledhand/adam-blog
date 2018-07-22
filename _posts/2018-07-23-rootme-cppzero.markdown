---
layout: post
title: Root-me) ELF C++ - 0 protection
date: 2018-07-23 07:10:20 +0900
description: # Add post description (optional)
img: rootme/c++zero/ptr.png # Add image post (optional)
tags: [CTF, Root-me]
author: purelledhand # Add name author (optional)
---

# 서론 : write-up

바이너리를 받고 IDA로 열고 싶었으나 IDA는 라업쓸때 캡쳐도 업로드해야하므로 GDB를 이용해서 분석했당


    purelledhand@purelledhand:~$ file ch25.bin
    ch25.bin: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24,

바이너리 파일은 ELF 32-bit 포맷이다.

    purelledhand@purelledhand:~$ ./ch25.bin AAAA
    Password incorrect.

인자값을 받고 바이너리 내 패스워드와 비교하는 바이너리로 추측이 되었다.

가장 먼저 disas main부터 해봤는데 비교분기문은 유일하게 하나만 있었다.

    0x08048b80 <+250>:	mov    eax,DWORD PTR [ebx+0x4]
    0x08048b83 <+253>:	add    eax,0x4
    0x08048b86 <+256>:	mov    eax,DWORD PTR [eax]
    0x08048b88 <+258>:	mov    DWORD PTR [esp+0x4],eax
    0x08048b8c <+262>:	lea    eax,[ebp-0x14]
    0x08048b8f <+265>:	mov    DWORD PTR [esp],eax
    0x08048b92 <+268>:	call   0x8048cf7 <_ZSteqIcSt11char_traitsIcESaIcEEbRKSbIT_T0_T1_EPKS3_>
    0x08048b97 <+273>:	test   al,al
    0x08048b99 <+275>:	je     0x8048be5 <main+351>
    0x08048b9b <+277>:	mov    DWORD PTR [esp+0x4],0x8048dfc
    0x08048ba3 <+285>:	mov    DWORD PTR [esp],0x804b100
    0x08048baa <+292>:	call   0x80487f0 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
    0x08048baf <+297>:	mov    DWORD PTR [esp+0x4],0x8048850
    0x08048bb7 <+305>:	mov    DWORD PTR [esp],eax
    0x08048bba <+308>:	call   0x8048840 <_ZNSolsEPFRSoS_E@plt>
    0x08048bbf <+313>:	mov    DWORD PTR [esp+0x4],0x8048e34
    0x08048bc7 <+321>:	mov    DWORD PTR [esp],0x804b100
    0x08048bce <+328>:	call   0x80487f0 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
    0x08048bd3 <+333>:	mov    DWORD PTR [esp+0x4],0x8048850
    0x08048bdb <+341>:	mov    DWORD PTR [esp],eax
    0x08048bde <+344>:	call   0x8048840 <_ZNSolsEPFRSoS_E@plt>

<+275>에서 보다시피 je를 수행하고 equal하다면 <+277>로 넘어가게 된다.

    (gdb) x/s 0x8048dfc
    0x8048dfc:	"Bravo, tu peux valider en utilisant ce mot de passe..."
    (gdb) x/s 0x8048e34
    0x8048e34:	"Congratz. You can validate with this password..."

그 아래 쭉 호출되는 함수들의 인자를 보면 위와 같이 성공 문자열이 담겨있었다.
검색해보니 _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc 시스템 콜은 C++의 cout를 의미하는 것 같당

성공문자열을 확인한 후 je 이전의 호출되는 함수를 분석해보았다.

    0x08048b8c <+262>:	lea    eax,[ebp-0x14]
    0x08048b8f <+265>:	mov    DWORD PTR [esp],eax
    => 0x08048b92 <+268>:	call   0x8048cf7 <_ZSteqIcSt11char_traitsIcESaIcEEbRKSbIT_T0_T1_EPKS3_>

ebp-0x14에 위치한 값을 eax에 넣고, 그 값을 esp가 있는 곳에 넣은 후 함수를 호출한다.
각 값들은 다음과 같다.
 

    (gdb) x/4wx $esp
    0xffffd5b0:	0xffffd5c4	0xffffd7cf	0xffffd5cc	0x08048d72
    (gdb) x/4wx $eax
    0xffffd5c4:	0x08050b24	0x08050a34	0x08050a1c	0xffffd5f0\

    (gdb) x/wx 0xffffd5c4
    0xffffd5c4:	0x08050b24
    (gdb) x/s 0x08050b24
    0x8050b24:	"Here_you_have_to_understand_a_little_C++_stuffs"

    (gdb) x/wx 0xffffd7cf
    0xffffd7cf:	0x41414141
    

보면 esp에 인자를 차례차례 넣고 함수를 호출했음을 알 수 있다.
argv값은 0xffffd7cf에 저장되었고, 이 값와 0xffffd5c4에 있는 문자열을 비교하는 함수였다. 그러므로 password는 **Here_you_have_to_understand_a_little_C++_stuffs**가 되는 것이다.


# 본론 : 내가 배운 건

조건점프 어셈을 오랜만에 보니깐 헷갈렸는데 이참에 다시 정리하고 갈 수 있어서 좋았다. IDA로도 이 바이너리를 열어봤는데 아직은 IDA보다 gdb가 편한것같다.. 열심히 해서 IDA도 익숙해져야겠당!

바이너리에서 password와 argv를 비교하는 함수부분을 다시 보자

    0x08048b8f <+265>:	mov    DWORD PTR [esp],eax
    => 0x08048b92 <+268>:	call   0x8048cf7 <_ZSteqIcSt11char_traitsIcESaIcEEbRKSbIT_T0_T1_EPKS3_>

이렇게 esp에 eax 주소값을 넣은 후 해당 함수를 호출한다.
esp와 eax의 값을 보면 아래와 같다.

    (gdb) x/4wx $esp
    0xffffd5b0:	0xffffd5c4	0xffffd7cf	0xffffd5cc	0x08048d72
    (gdb) x/4wx $eax
    0xffffd5c4:	0x08050b24	0x08050a34	0x08050a1c	0xffffd5f0\

    (gdb) x/wx 0xffffd5c4
    0xffffd5c4:	0x08050b24
    (gdb) x/s 0x08050b24
    0x8050b24:	"Here_you_have_to_understand_a_little_C++_stuffs"

이 구조를 시각화한다면 아래 그림과 비슷할거라고 생각된다. 

<img src="{{site.baseurl}}/assets/img/rootme/c++zero/ptr.png" alt="ptr" style="width: 500px; margin: 0 auto;"/>

이 흐름을 이해 한 후 메모리구조를 그려보면 다음과 같다.

<img src="{{site.baseurl}}/assets/img/rootme/c++zero/stack.png" alt="stack" style="width: 400px; margin: 0 auto;"/>


# 결론

> 1. 분기문을 기준으로 디버깅
> 2. 32bit 바이너리의 레지스터, 메모리 구조 체계를 정리할 수 있었다.
> 3. 시간나면 C++을 공부해보장