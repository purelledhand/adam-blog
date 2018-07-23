---
layout: post
title: Root-me) ELF - Fake instruction
date: 2018-07-23 08:49:20 +0900
description: # Add post description (optional)
img: post-2.jpg # Add image post (optional)
tags: [CTF, Root-me]
author: purelledhand # Add name author (optional)
---

# 서론 : write-up

    purelledhand@purelledhand:~$ ./crackme AA
    Verification de votre mot de passe..
    le voie de la sagesse te guidera, tache de trouver le mot de passe petit padawaan 


32-bit ELF이다. 인자값을 받고 바이너리 내 패스워드와 비교하는 바이너리로 추측된다. 한국어로 번역해보면

> 암호를 확인하고 있습니다 ..
> 지혜의 길은 당신을 이끌 것입니다 작은 팟도완 암호를 찾기 위해 장소

그렇다고 한다..

IDA를 통해 로직을 따라가보면서 main에서 WPA함수를 호출하는 것을 확인 했다.

WPA함수는 아래와 같다.

    oid __cdecl __noreturn WPA(char *s1, char *s2)
    {
    s2[11] = 13;
    s2[12] = 10;
    puts(s);
    if ( !strcmp(s1, s2) )
        blowfish();
    RS4();
    }

s1 == s2 이면 blowfish() 호출
s1 != s2 이면 RS4() 호출

이 때 RS4()는 아무값과 함께 바이너리를 실행시켰을 때 출력됐던 문자열과 같은 문자열을 출력하고 바이너리를 종료시키는 함수였다.

    void __noreturn RS4()
    {
    printf("le voie de la sagesse te guidera, tache de trouver le mot de passe petit padawaan \n ");
    exit(0);
    }

이를 통해서 blowfish() 함수가 비밀번호가 일치했을 때 실행되는 함수임이 확실시 되어 blowfish() 함수를 분석했다.

    void __noreturn blowfish()
    {
    char *v0; // ST14_4
    unsigned int v1; // [esp+Ch] [ebp-4Ch]
    int v2; // [esp+18h] [ebp-40h]
    int v3; // [esp+1Ch] [ebp-3Ch]
    int v4; // [esp+20h] [ebp-38h]
    char v5[23]; // [esp+24h] [ebp-34h]
    int v6; // [esp+3Bh] [ebp-1Dh]
    int v7; // [esp+3Fh] [ebp-19h]
    int v8; // [esp+43h] [ebp-15h]
    int v9; // [esp+47h] [ebp-11h]
    int v10; // [esp+4Bh] [ebp-Dh]
    int v11; // [esp+4Fh] [ebp-9h]
    char v12; // [esp+53h] [ebp-5h]
    unsigned int v13; // [esp+54h] [ebp-4h]

    v13 = __readgsdword(0x14u);
    v2 = 1700948332;
    v3 = -1446808462;
    v4 = 33;
    v1 = 0;
    do
    {
        *(_DWORD *)&v5[v1] = 0;
        v1 += 4;
    }
    while ( v1 < 0x14 );
    v0 = &v5[v1];
    *(_WORD *)v0 = 0;
    v0[2] = 0;
    v6 = 1197682783;
    v7 = 1832215402;
    v8 = 1412771423;
    v9 = 948421427;
    v10 = -1020245437;
    v11 = 961034624;
    v12 = 0;
    printf(aAuthentificati, &v2);
    exit(0);
    }

보이는 것처럼 무의미한? 연산들이 쭈루룩 나오고 마지막에 printf() 함수를 호출한 후 바이너리를 종료시킨다.

GDB에서 blowfish() 함수를 실행시켜 보았다.

    (gdb) run AAAA
    Starting program: /home/purelledhand/crackme AAAA

    Breakpoint 3, 0x08048578 in main ()

    (gdb) set $eip = 0x804872c
    (gdb) c
    Continuing.
    '+) Authentification réussie...
    U'r root! 

    sh 3.0 # password: liberté!

문제를 풀면서 파악한 함수흐름은 아래와 같다.

<img src="{{site.baseurl}}/assets/img/rootme/cracking/fake.png" alt="flow" style="width: 500px; margin: 0 auto;"/>


음 근데 문제 제목이 fake instruction인 이유는 무의미한 연산들을 많이 넣어놔서 그런건지.. 풀긴 풀었는데 제목의 이유를 모르니까 문제의 요지를 모르는 것 같아서 찝찝하다ㅠㅠ

# 본론 : 내가 배운 건

gdb 내에서 eip를 포함한 레지스터 값을 변조 하는건 정말 신세계같다..

bof문제들 중 bof로 eip주소를 이동할 곳으로 덮는 게 필요없이 gdb에서 바로 eip값을 바꿀 수 있다니..

그동안 삽질하면서 풀었던것들이 허무하기도 하지만 삽질한만큼 대단하게 느껴지는 기능이었다..

# 결론

> 1. set $eip
> 2. pwndbg로 넘어가자