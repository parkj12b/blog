+++
title = "C99 표준 기능"
date = "2026-04-21T23:05:00Z"
draft = false
tags = ["C99", "C"]
categories = ["Programming"]
featureimage = "featured.svg"
+++

{{< linkcard 
    title="C99 - 위키백과" 
    description="C99 표준은 현대 C 개발에서 표준으로 사용되는 많은 기능들을 도입했습니다." 
    url="https://en.wikipedia.org/wiki/C99" 
>}}

{{< linkcard 
    title="Kconfig 언어 — 리눅스 커널 문서" 
    description="리눅스 커널 빌드 시스템에서 사용되는 Kconfig 언어에 대한 공식 문서입니다." 
    url="https://www.kernel.org/doc/html/latest/kbuild/kconfig-language.html" 
>}}

C로 개발할 때 모두가 C99를 잘 안다고 생각하지만, 정말 그럴까요? (글쎄요, 저는 아마 모르는 것 같습니다 Xd)

오늘은 표준에 익숙해지기 위해 C99의 몇 가지 기능을 살펴보겠습니다.

# Features

## inline

https://en.cppreference.com/c/language/inline

```c
inline const char *saddr(void) // the inline definition for use in this file
{
    static const char name[] = "saddr";
    return name;
}

int compare_name(void)
{
    return saddr() == saddr(); // unspecified behavior, one call could be external
}

extern const char *saddr(void); // an external definition is generated, too
```

- 함수를 호출하고 콜 스택을 추가하는 대신, inline 키워드는 함수가 호출되는 위치에 코드를 삽입하도록 컴파일러에 힌트를 주어 함수 호출 오버헤드를 줄입니다.
- 함수가 여러 곳에서 호출될 때 코드 크기가 커지는 단점이 있습니다.
    - 임베디드 시스템에서는 이런 이유로 대신 `define`을 사용한다고 들은 것 같습니다.
- 같은 파일에서 호출할 때 static으로 만들지 않으면 찾을 수 없습니다.
    - C에서 함수는 기본적으로 extern이므로, 다른 파일에서 선언할 때 extern을 추가할 필요가 없습니다.
    - static = local
- -O0 사용 시 inline을 무시합니다.
- -O0에서 inline을 강제하려면 gcc의 힘을 빌려야 합니다.
    - __attribute__((always_inline)) static inline void add_one_inline(int *x)
- 컴파일러가 보통 적용 가능한 경우 언제든지 inline을 추가하므로, 정말 무엇을 하고 있는지 잘 알지 못한다면 inline을 건드리지 않는 것이 좋습니다.

### disassembly (AT&T)

```nasm
0000000000001189 <add_one>:
    1189:	f3 0f 1e fa          	endbr64
    118d:	55                   	push   %rbp
    118e:	48 89 e5             	mov    %rsp,%rbp
    1191:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
    1195:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
    1199:	8b 00                	mov    (%rax),%eax
    119b:	8d 50 01             	lea    0x1(%rax),%edx
    119e:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
    11a2:	89 10                	mov    %edx,(%rax)
    11a4:	90                   	nop
    11a5:	5d                   	pop    %rbp
    11a6:	c3                   	ret

00000000000011a7 <get_time_in_seconds>:
    11a7:	f3 0f 1e fa          	endbr64
    11ab:	55                   	push   %rbp
    11ac:	48 89 e5             	mov    %rsp,%rbp
    11af:	48 83 ec 20          	sub    $0x20,%rsp
    11b3:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
    11ba:	00 00 
    11bc:	48 89 45 f8          	mov    %rax,-0x8(%rbp)
    11c0:	31 c0                	xor    %eax,%eax
    11c2:	48 8d 45 e0          	lea    -0x20(%rbp),%rax
    11c6:	48 89 c6             	mov    %rax,%rsi
    11c9:	bf 01 00 00 00       	mov    $0x1,%edi
    11ce:	e8 9d fe ff ff       	call   1070 <clock_gettime@plt>
    11d3:	48 8b 45 e0          	mov    -0x20(%rbp),%rax
    11d7:	66 0f ef c9          	pxor   %xmm1,%xmm1
    11db:	f2 48 0f 2a c8       	cvtsi2sd %rax,%xmm1
    11e0:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
    11e4:	66 0f ef c0          	pxor   %xmm0,%xmm0
    11e8:	f2 48 0f 2a c0       	cvtsi2sd %rax,%xmm0
    11ed:	f2 0f 10 15 a3 0e 00 	movsd  0xea3(%rip),%xmm2        # 2098 <_IO_stdin_used+0x98>
    11f4:	00 
    11f5:	f2 0f 5e c2          	divsd  %xmm2,%xmm0
    11f9:	f2 0f 58 c1          	addsd  %xmm1,%xmm0
    11fd:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
    1201:	64 48 2b 04 25 28 00 	sub    %fs:0x28,%rax
    1208:	00 00 
    120a:	74 05                	je     1211 <get_time_in_seconds+0x6a>
    120c:	e8 6f fe ff ff       	call   1080 <__stack_chk_fail@plt>
    1211:	c9                   	leave
    1212:	c3                   	ret

0000000000001213 <main>:
    1213:	f3 0f 1e fa          	endbr64
    1217:	55                   	push   %rbp
    1218:	48 89 e5             	mov    %rsp,%rbp
    121b:	48 83 ec 30          	sub    $0x30,%rsp
    121f:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
    1226:	00 00 
    1228:	48 89 45 f8          	mov    %rax,-0x8(%rbp)
    122c:	31 c0                	xor    %eax,%eax
    122e:	c7 45 d0 00 00 00 00 	movl   $0x0,-0x30(%rbp)
    1235:	c7 45 d4 00 00 00 00 	movl   $0x0,-0x2c(%rbp)
    123c:	be 80 96 98 00       	mov    $0x989680,%esi
    1241:	48 8d 05 c0 0d 00 00 	lea    0xdc0(%rip),%rax        # 2008 <_IO_stdin_used+0x8>
    1248:	48 89 c7             	mov    %rax,%rdi
    124b:	b8 00 00 00 00       	mov    $0x0,%eax
    1250:	e8 3b fe ff ff       	call   1090 <printf@plt>
    1255:	b8 00 00 00 00       	mov    $0x0,%eax
    125a:	e8 48 ff ff ff       	call   11a7 <get_time_in_seconds>
    125f:	66 48 0f 7e c0       	movq   %xmm0,%rax
    1264:	48 89 45 e0          	mov    %rax,-0x20(%rbp)
    1268:	c7 45 d8 00 00 00 00 	movl   $0x0,-0x28(%rbp)
    126f:	eb 10                	jmp    1281 <main+0x6e>
    1271:	48 8d 45 d0          	lea    -0x30(%rbp),%rax
    1275:	48 89 c7             	mov    %rax,%rdi
    1278:	e8 0c ff ff ff       	call   1189 <add_one>
    127d:	83 45 d8 01          	addl   $0x1,-0x28(%rbp)
    1281:	81 7d d8 7f 96 98 00 	cmpl   $0x98967f,-0x28(%rbp)
    1288:	7e e7                	jle    1271 <main+0x5e>
    128a:	b8 00 00 00 00       	mov    $0x0,%eax
    128f:	e8 13 ff ff ff       	call   11a7 <get_time_in_seconds>
    1294:	66 48 0f 7e c0       	movq   %xmm0,%rax
    1299:	48 89 45 e8          	mov    %rax,-0x18(%rbp)
    129d:	f2 0f 10 45 e8       	movsd  -0x18(%rbp),%xmm0
    12a2:	f2 0f 5c 45 e0       	subsd  -0x20(%rbp),%xmm0
    12a7:	66 48 0f 7e c0       	movq   %xmm0,%rax
    12ac:	66 48 0f 6e c0       	movq   %rax,%xmm0
    12b1:	48 8d 05 7a 0d 00 00 	lea    0xd7a(%rip),%rax        # 2032 <_IO_stdin_used+0x32>
    12b8:	48 89 c7             	mov    %rax,%rdi
    12bb:	b8 01 00 00 00       	mov    $0x1,%eax
    12c0:	e8 cb fd ff ff       	call   1090 <printf@plt>
    12c5:	b8 00 00 00 00       	mov    $0x0,%eax
    12ca:	e8 d8 fe ff ff       	call   11a7 <get_time_in_seconds>
    12cf:	66 48 0f 7e c0       	movq   %xmm0,%rax
    12d4:	48 89 45 e0          	mov    %rax,-0x20(%rbp)
    12d8:	c7 45 dc 00 00 00 00 	movl   $0x0,-0x24(%rbp)
    12df:	eb 1c                	jmp    12fd <main+0xea>
    12e1:	48 8d 45 d4          	lea    -0x2c(%rbp),%rax
    12e5:	48 89 45 f0          	mov    %rax,-0x10(%rbp)
    12e9:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
    12ed:	8b 00                	mov    (%rax),%eax
    12ef:	8d 50 01             	lea    0x1(%rax),%edx
    12f2:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
    12f6:	89 10                	mov    %edx,(%rax)
    12f8:	90                   	nop
    12f9:	83 45 dc 01          	addl   $0x1,-0x24(%rbp)
    12fd:	81 7d dc 7f 96 98 00 	cmpl   $0x98967f,-0x24(%rbp)
    1304:	7e db                	jle    12e1 <main+0xce>
    1306:	b8 00 00 00 00       	mov    $0x0,%eax
    130b:	e8 97 fe ff ff       	call   11a7 <get_time_in_seconds>
    1310:	66 48 0f 7e c0       	movq   %xmm0,%rax
    1315:	48 89 45 e8          	mov    %rax,-0x18(%rbp)
    1319:	f2 0f 10 45 e8       	movsd  -0x18(%rbp),%xmm0
    131e:	f2 0f 5c 45 e0       	subsd  -0x20(%rbp),%xmm0
    1323:	66 48 0f 7e c0       	movq   %xmm0,%rax
    1328:	66 48 0f 6e c0       	movq   %rax,%xmm0
    132d:	48 8d 05 19 0d 00 00 	lea    0xd19(%rip),%rax        # 204d <_IO_stdin_used+0x4d>
    1334:	48 89 c7             	mov    %rax,%rdi
    1337:	b8 01 00 00 00       	mov    $0x1,%eax
    133c:	e8 4f fd ff ff       	call   1090 <printf@plt>
    1341:	8b 55 d0             	mov    -0x30(%rbp),%edx
    1344:	8b 45 d4             	mov    -0x2c(%rbp),%eax
    1347:	01 d0                	add    %edx,%eax
    1349:	89 05 c5 2c 00 00    	mov    %eax,0x2cc5(%rip)        # 4014 <global_sink>
    134f:	8b 05 bf 2c 00 00    	mov    0x2cbf(%rip),%eax        # 4014 <global_sink>
    1355:	89 c6                	mov    %eax,%esi
    1357:	48 8d 05 0a 0d 00 00 	lea    0xd0a(%rip),%rax        # 2068 <_IO_stdin_used+0x68>
    135e:	48 89 c7             	mov    %rax,%rdi
    1361:	b8 00 00 00 00       	mov    $0x0,%eax
    1366:	e8 25 fd ff ff       	call   1090 <printf@plt>
    136b:	b8 00 00 00 00       	mov    $0x0,%eax
    1370:	48 8b 55 f8          	mov    -0x8(%rbp),%rdx
    1374:	64 48 2b 14 25 28 00 	sub    %fs:0x28,%rdx
    137b:	00 00 
    137d:	74 05                	je     1384 <main+0x171>
    137f:	e8 fc fc ff ff       	call   1080 <__stack_chk_fail@plt>
    1384:	c9                   	leave
    1385:	c3                   	ret
```

### disassembly (Intel)

```nasm
0000000000001189 <add_one>:
    1189:	f3 0f 1e fa          	endbr64
    118d:	55                   	push   rbp
    118e:	48 89 e5             	mov    rbp,rsp
    1191:	48 89 7d f8          	mov    QWORD PTR [rbp-0x8],rdi
    1195:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
    1199:	8b 00                	mov    eax,DWORD PTR [rax]
    119b:	8d 50 01             	lea    edx,[rax+0x1]
    119e:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
    11a2:	89 10                	mov    DWORD PTR [rax],edx
    11a4:	90                   	nop
    11a5:	5d                   	pop    rbp
    11a6:	c3                   	ret

00000000000011a7 <get_time_in_seconds>:
    11a7:	f3 0f 1e fa          	endbr64
    11ab:	55                   	push   rbp
    11ac:	48 89 e5             	mov    rbp,rsp
    11af:	48 83 ec 20          	sub    rsp,0x20
    11b3:	64 48 8b 04 25 28 00 	mov    rax,QWORD PTR fs:0x28
    11ba:	00 00 
    11bc:	48 89 45 f8          	mov    QWORD PTR [rbp-0x8],rax
    11c0:	31 c0                	xor    eax,eax
    11c2:	48 8d 45 e0          	lea    rax,[rbp-0x20]
    11c6:	48 89 c6             	mov    rsi,rax
    11c9:	bf 01 00 00 00       	mov    edi,0x1
    11ce:	e8 9d fe ff ff       	call   1070 <clock_gettime@plt>
    11d3:	48 8b 45 e0          	mov    rax,QWORD PTR [rbp-0x20]
    11d7:	66 0f ef c9          	pxor   xmm1,xmm1
    11db:	f2 48 0f 2a c8       	cvtsi2sd xmm1,rax
    11e0:	48 8b 45 e8          	mov    rax,QWORD PTR [rbp-0x18]
    11e4:	66 0f ef c0          	pxor   xmm0,xmm0
    11e8:	f2 48 0f 2a c0       	cvtsi2sd xmm0,rax
    11ed:	f2 0f 10 15 a3 0e 00 	movsd  xmm2,QWORD PTR [rip+0xea3]        # 2098 <_IO_stdin_used+0x98>
    11f4:	00 
    11f5:	f2 0f 5e c2          	divsd  xmm0,xmm2
    11f9:	f2 0f 58 c1          	addsd  xmm0,xmm1
    11fd:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
    1201:	64 48 2b 04 25 28 00 	sub    rax,QWORD PTR fs:0x28
    1208:	00 00 
    120a:	74 05                	je     1211 <get_time_in_seconds+0x6a>
    120c:	e8 6f fe ff ff       	call   1080 <__stack_chk_fail@plt>
    1211:	c9                   	leave
    1212:	c3                   	ret

0000000000001213 <main>:
    1213:	f3 0f 1e fa          	endbr64
    1217:	55                   	push   rbp
    1218:	48 89 e5             	mov    rbp,rsp
    121b:	48 83 ec 30          	sub    rsp,0x30
    121f:	64 48 8b 04 25 28 00 	mov    rax,QWORD PTR fs:0x28
    1226:	00 00 
    1228:	48 89 45 f8          	mov    QWORD PTR [rbp-0x8],rax
    122c:	31 c0                	xor    eax,eax
    122e:	c7 45 d0 00 00 00 00 	mov    DWORD PTR [rbp-0x30],0x0
    1235:	c7 45 d4 00 00 00 00 	mov    DWORD PTR [rbp-0x2c],0x0
    123c:	be 80 96 98 00       	mov    esi,0x989680
    1241:	48 8d 05 c0 0d 00 00 	lea    rax,[rip+0xdc0]        # 2008 <_IO_stdin_used+0x8>
    1248:	48 89 c7             	mov    rdi,rax
    124b:	b8 00 00 00 00       	mov    eax,0x0
    1250:	e8 3b fe ff ff       	call   1090 <printf@plt>
    1255:	b8 00 00 00 00       	mov    eax,0x0
    125a:	e8 48 ff ff ff       	call   11a7 <get_time_in_seconds>
    125f:	66 48 0f 7e c0       	movq   rax,xmm0
    1264:	48 89 45 e0          	mov    QWORD PTR [rbp-0x20],rax
    1268:	c7 45 d8 00 00 00 00 	mov    DWORD PTR [rbp-0x28],0x0
    126f:	eb 10                	jmp    1281 <main+0x6e>
    1271:	48 8d 45 d0          	lea    rax,[rbp-0x30]
    1275:	48 89 c7             	mov    rdi,rax
    1278:	e8 0c ff ff ff       	call   1189 <add_one>
    127d:	83 45 d8 01          	add    DWORD PTR [rbp-0x28],0x1
    1281:	81 7d d8 7f 96 98 00 	cmp    DWORD PTR [rbp-0x28],0x98967f
    1288:	7e e7                	jle    1271 <main+0x5e>
    128a:	b8 00 00 00 00       	mov    eax,0x0
    128f:	e8 13 ff ff ff       	call   11a7 <get_time_in_seconds>
    1294:	66 48 0f 7e c0       	movq   rax,xmm0
    1299:	48 89 45 e8          	mov    QWORD PTR [rbp-0x18],rax
    129d:	f2 0f 10 45 e8       	movsd  xmm0,QWORD PTR [rbp-0x18]
    12a2:	f2 0f 5c 45 e0       	subsd  xmm0,QWORD PTR [rbp-0x20]
    12a7:	66 48 0f 7e c0       	movq   rax,xmm0
    12ac:	66 48 0f 6e c0       	movq   xmm0,rax
    12b1:	48 8d 05 7a 0d 00 00 	lea    rax,[rip+0xd7a]        # 2032 <_IO_stdin_used+0x32>
    12b8:	48 89 c7             	mov    rdi,rax
    12bb:	b8 01 00 00 00       	mov    eax,0x1
    12c0:	e8 cb fd ff ff       	call   1090 <printf@plt>
    12c5:	b8 00 00 00 00       	mov    eax,0x0
    12ca:	e8 d8 fe ff ff       	call   11a7 <get_time_in_seconds>
    12cf:	66 48 0f 7e c0       	movq   rax,xmm0
    12d4:	48 89 45 e0          	mov    QWORD PTR [rbp-0x20],rax
    12d8:	c7 45 dc 00 00 00 00 	mov    DWORD PTR [rbp-0x24],0x0
    12df:	eb 1c                	jmp    12fd <main+0xea>
    12e1:	48 8d 45 d4          	lea    rax,[rbp-0x2c]
    12e5:	48 89 45 f0          	mov    QWORD PTR [rbp-0x10],rax
    12e9:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
    12ed:	8b 00                	mov    eax,DWORD PTR [rax]
    12ef:	8d 50 01             	lea    edx,[rax+0x1]
    12f2:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
    12f6:	89 10                	mov    DWORD PTR [rax],edx
    12f8:	90                   	nop
    12f9:	83 45 dc 01          	add    DWORD PTR [rbp-0x24],0x1
    12fd:	81 7d dc 7f 96 98 00 	cmp    DWORD PTR [rbp-0x24],0x98967f
    1304:	7e db                	jle    12e1 <main+0xce>
    1306:	b8 00 00 00 00       	mov    eax,0x0
    130b:	e8 97 fe ff ff       	call   11a7 <get_time_in_seconds>
    1310:	66 48 0f 7e c0       	movq   rax,xmm0
    1315:	48 89 45 e8          	mov    QWORD PTR [rbp-0x18],rax
    1319:	f2 0f 10 45 e8       	movsd  xmm0,QWORD PTR [rbp-0x18]
    131e:	f2 0f 5c 45 e0       	subsd  xmm0,QWORD PTR [rbp-0x20]
    1323:	66 48 0f 7e c0       	movq   rax,xmm0
    1328:	66 48 0f 6e c0       	movq   xmm0,rax
    132d:	48 8d 05 19 0d 00 00 	lea    rax,[rip+0xd19]        # 204d <_IO_stdin_used+0x4d>
    1334:	48 89 c7             	mov    rdi,rax
    1337:	b8 01 00 00 00       	mov    eax,0x1
    133c:	e8 4f fd ff ff       	call   1090 <printf@plt>
    1341:	8b 55 d0             	mov    edx,DWORD PTR [rbp-0x30]
    1344:	8b 45 d4             	mov    eax,DWORD PTR [rbp-0x2c]
    1347:	01 d0                	add    eax,edx
    1349:	89 05 c5 2c 00 00    	mov    DWORD PTR [rip+0x2cc5],eax        # 4014 <global_sink>
    134f:	8b 05 bf 2c 00 00    	mov    eax,DWORD PTR [rip+0x2cbf]        # 4014 <global_sink>
    1355:	89 c6                	mov    esi,eax
    1357:	48 8d 05 0a 0d 00 00 	lea    rax,[rip+0xd0a]        # 2068 <_IO_stdin_used+0x68>
    135e:	48 89 c7             	mov    rdi,rax
    1361:	b8 00 00 00 00       	mov    eax,0x0
    1366:	e8 25 fd ff ff       	call   1090 <printf@plt>
    136b:	b8 00 00 00 00       	mov    eax,0x0
    1370:	48 8b 55 f8          	mov    rdx,QWORD PTR [rbp-0x8]
    1374:	64 48 2b 14 25 28 00 	sub    rdx,QWORD PTR fs:0x28
    137b:	00 00 
    137d:	74 05                	je     1384 <main+0x171>
    137f:	e8 fc fc ff ff       	call   1080 <__stack_chk_fail@plt>
    1384:	c9                   	leave
    1385:	c3                   	ret
```

- Disassembly 결과 add_one_inline에 대한 심볼이 보이지 않습니다.

![image.png](image.png)

### Decl intermingled with code

```c
...
printf("Hello World!");
int a = 10; //decl intermingled with code

for (int i = 0; i < a; i++) //int i inter mingled with for loop
	...
```

- 이게 C99 이후에나 가능했다는 사실을 몰랐네요 😲

## _Bool / stdbool.h

```c
#include <stdio.h>
#include <stdbool.h>

int main() {
	_Bool is_today_a_good_day = 1;

	if (is_today_a_good_day)
		printf("Today is a good day :)\n");

	bool is_today_also_a_good_day = true;

	if (is_today_also_a_good_day)
		printf("Today is also a good day :)\n");
}
```

- 꽤 직관적입니다.

### stdbool.h

```c
#ifndef __STDBOOL_H
#define __STDBOOL_H

#define __bool_true_false_are_defined 1

#if defined(__STDC_VERSION__) && __STDC_VERSION__ > 201710L
/* FIXME: We should be issuing a deprecation warning here, but cannot yet due
 * to system headers which include this header file unconditionally.
 */
#elif !defined(__cplusplus)
#define bool _Bool
#define true 1
#define false 0
#elif defined(__GNUC__) && !defined(__STRICT_ANSI__)
/* Define _Bool as a GNU extension. */
#define _Bool bool
#if defined(__cplusplus) && __cplusplus < 201103L
/* For C++98, define bool, false, true as a GNU extension. */
#define bool bool
#define false false
#define true true
#endif
#endif

#endif /* __STDBOOL_H */

```

## Complex type

```c
#include <stdio.h>
#include <complex.h>

int main() {
    // 1. Declaration and Initialization
    // 'I' is a macro defined in complex.h representing the imaginary unit
    double complex z1 = 3.0 + 4.0 * I;
    double complex z2 = 1.0 - 2.0 * I;

    // 2. Native Arithmetic
    // You can use standard operators (+, -, *, /) directly on complex types
    double complex sum = z1 + z2;
    double complex diff = z1 - z2;
    double complex product = z1 * z2;

    // 3. Printing Complex Numbers
    // printf doesn't have a direct format specifier for complex numbers,
    // so you extract the real and imaginary parts using creal() and cimag()
    printf("z1      = %.1f %+.1fi\n", creal(z1), cimag(z1));
    printf("z2      = %.1f %+.1fi\n", creal(z2), cimag(z2));
    
    printf("\n--- Results ---\n");
    printf("Sum     = %.1f %+.1fi\n", creal(sum), cimag(sum));
    printf("Diff    = %.1f %+.1fi\n", creal(diff), cimag(diff));
    printf("Product = %.1f %+.1fi\n", creal(product), cimag(product));

    return 0;
}
```

![image.png](image%201.png)

## Flexible array members

```c
#include <stdio.h>
#include <stdlib.h>

// --- Approach 1: Struct with a Pointer ---
struct PointerVector {
    size_t length;
    int *data;
};

// --- Approach 2: Struct with a Flexible Array ---
struct FlexibleVector {
    size_t length;
    int data[]; // Flexible array must be the last member
};

int main() {
    size_t num_items = 5;

    printf("=== Approach 1: Pointer Member (2 Mallocs) ===\n");
    
    // Allocation
    struct PointerVector *p_vec = malloc(sizeof(struct PointerVector));
    if (!p_vec) return 1;
    p_vec->length = num_items;
    p_vec->data = malloc(num_items * sizeof(int));
    if (!p_vec->data) { free(p_vec); return 1; }

    // Print sizes and memory layout
    printf("Reported struct size: %zu bytes\n", sizeof(struct PointerVector));
    printf("Struct is located at: %p\n", (void*)p_vec);
    printf("Data is located at:   %p  <-- Notice the massive jump in memory!\n", (void*)p_vec->data);

    // Cleanup (Order matters!)
    free(p_vec->data);
    free(p_vec);

    printf("\n--------------------------------------------------\n\n");

    printf("=== Approach 2: Flexible Array (1 Malloc) ===\n");

    // Allocation
    size_t total_size = sizeof(struct FlexibleVector) + (num_items * sizeof(int));
    struct FlexibleVector *f_vec = malloc(total_size);
    if (!f_vec) return 1;
    f_vec->length = num_items;

    // Print sizes and memory layout
    printf("Reported struct size: %zu bytes (Notice it's smaller!)\n", sizeof(struct FlexibleVector));
    printf("Struct is located at: %p\n", (void*)f_vec);
    printf("Data is located at:   %p  <-- Notice it is immediately after the struct!\n", (void*)f_vec->data);

    // Cleanup (Just one call)
    free(f_vec);

    return 0;
}
```

### Flexible Array Members의 3가지 황금 규칙

이것을 사용하려면 컴파일러는 다음 세 가지 엄격한 규칙을 따를 것을 요구합니다:

1. **마지막에 위치해야 함:** `struct`의 맨 마지막 변수에만 `[]`를 붙일 수 있습니다.
2. **혼자 있을 수 없음:** `struct`는 flexible array 앞에 적어도 하나 이상의 다른 표준(이름이 있는) 멤버를 포함해야 합니다. `int data[]`만 포함하는 struct는 만들 수 없습니다.
3. **`sizeof()`가 이를 무시함:** `sizeof(struct DynamicVector)`를 실행하면 `length` 변수의 크기(및 패딩)만 반환합니다. Flexible array가 전혀 존재하지 않는 것처럼 크기를 계산합니다. 그렇기 때문에 `malloc` 호출 시 직접 배열 크기를 추가해야 합니다.

### Linux Kernel 예시

```c
struct inotify_event {
    __s32           wd;             /* watch descriptor */
    __u32           mask;           /* watch mask */
    __u32           cookie;         /* cookie to synchronize two events */
    __u32           len;            /* length (including nulls) of name */
    char            name[];         /* stub for possible name */
};
```

- Linux Kernel은 이 기법을 꽤 자주 사용합니다. (적어도 제가 본 것들 중에서는요 Xd)
- 만약 name이 char * 였다면:
    - `struct inotify_event` 할당
    - `char *name` 할당
    - `name`을 `struct inotify_event`에 할당
- Flexible Array Member를 사용하면:
    - alloc(sizeof(inotify_even) + /* size of name */)
    - 한 번에 할당 가능합니다. 포인터를 따로 전달할 필요도 없죠.

멋진 트릭입니다!

## tgmath.h

```c
float res1 = sqrt(f);  // The compiler silently translates this to sqrtf(f)
    double res2 = sqrt(d); // Translates to standard sqrt(d)
    long double res3 = sqrt(ld); // Translates to sqrtl(ld)
    
    double complex res4 = sqrt(z); // Translates to csqrt(z)

    printf("Float root:  %.1f\n", res1);
    printf("Double root: %.1f\n", res2);
    printf("Long root:   %.1Lf\n", res3);
    printf("Complex root: %.1f %+.1fi\n", creal(res4), cimag(res4));
```

- 컴파일러가 조용히 sqrt를 sqrtf, sqrtl, csqrt로 변환합니다.
- 아무도 이것을 사용하는지 모르겠네요.

[The ugliest C feature: <tgmath.h>](https://www.reddit.com/r/programming/comments/1maa65/the_ugliest_c_feature_tgmathh/)

- gg

## Designated Initializers

```c
#include <stdio.h>

struct point {
    int x;
    int y;
    int z;
};

int main() {
    // Designated initializers for structures
    struct point p = { .y = 2, .x = 1, .z = 3 };

    // Designated initializers for arrays
    int arr[10] = { [2] = 20, [5] = 50, [8] = 80 };

    printf("Point: x=%d, y=%d, z=%d\n", p.x, p.y, p.z);
    
    printf("Array elements:\n");
    for (int i = 0; i < 10; i++) {
        if (arr[i] != 0) {
            printf("arr[%d] = %d\n", i, arr[i]);
        }
    }

    return 0;
}
```

## Compound Literal

```c
#include <stdio.h>

struct Point {
    int x;
    int y;
};

// A simple function that expects a struct
void draw_point(struct Point p) {
    printf("Drawing at X:%d, Y:%d\n", p.x, p.y);
}

// A function that expects a POINTER to a struct
void move_point(struct Point *p, int dx, int dy) {
    p->x += dx;
    p->y += dy;
    printf("Moved to X:%d, Y:%d\n", p->x, p->y);
}

int main() {
    // --- 이전 방식 (C89) ---
    // 한 번만 사용하는 변수로 스코프를 오염시켜야 합니다.
    struct Point temp_p = {10, 20};
    draw_point(temp_p);

    // --- 새로운 방식 (Compound Literal) ---
    // 함수 호출 내부에서 직접 이름 없는 struct를 생성합니다!
    draw_point((struct Point){30, 40});

    // --- "궁극의" 방식 ---
    // Compound literals와 designated initializers를 결합합니다.
    draw_point((struct Point){ .y = 100, .x = 50 });

    // --- 마법 같은 트릭 (주소 가져오기) ---
    // 표준 리터럴(예: 숫자 5)과 달리 compound literals는 실제로 메모리에 존재합니다.
    // 즉, '&'를 사용하여 주소를 가져올 수 있습니다!
    move_point(&(struct Point){1, 1}, 5, 5);

    // --- 즉석 배열 ---
    // 변수를 선언하지 않고 전체 배열을 전달할 수 있습니다!
    print_sum((int[]){10, 20, 30, 40, 50}, 5);

    return 0;
}
```

### Compound Literals의 3가지 황금 규칙

1. **L-values임:** 캐스트(예: `(int)3.14`)와 달리, compound literal은 실제로 RAM에 실제 객체를 생성합니다. 즉, `&`를 사용하여 메모리 주소를 얻거나 내용을 수정하거나 포인터로 전달할 수 있습니다.
2. **스코프 적용:** 임시 객체는 정의된 블록의 스코프에서 생성됩니다. 블록이 끝나면 파괴됩니다. 함수에서 compound literal에 대한 포인터를 반환하지 **마세요**!
    
    ```c
    // 위험! 함수가 반환되면 배열이 사라집니다.
    int* bad_function() {
        return (int[]){1, 2, 3}; 
    }
    ```
    
3. **읽기 전용은 선택사항:** `const` 키워드를 추가하면 컴파일러가 문자열 리터럴처럼 읽기 전용 메모리에 배치하여 최적화할 수 있습니다.
    
    ```c
    draw_point((const struct Point){10, 20});
    ```
    

## Variadic Macros

```c
#include <stdio.h>

// '...'은 이 매크로가 추가 인자를 받는다는 것을 의미합니다.
// __VA_ARGS__는 그 추가 인자들이 무엇이든 그 인자들로 대체됩니다.
#define LOG_INFO(format, ...) printf("[INFO] " format "\n", __VA_ARGS__)

int main() {
    int users = 5;
    char *system = "Database";

    // 다음과 같이 확장됩니다: printf("[INFO] " "System %s has %d users." "\n", system, users)
    LOG_INFO("System %s has %d users.", system, users);

    return 0;
}
```

- 이는 Kernel에서 자주 사용됩니다.

## restrict keyword

이는 제가 42seoul 여정 중에 처음 마주한 키워드 중 하나입니다.

```c
#include <string.h>

void *memcpy(size_t n;
						 void dest[restrict n],
						 const void src[restrict n], size_t n);
```

- C에는 포인터 에일리어싱(aliasing)이 있기 때문에 dest와 src가 겹치는 메모리 위치를 가리킬 수 있어 가비지 값이 복사될 수 있습니다.

```c
void add_stuff(int *a, int *b, int *out) {
    *out = *a + 10;
    *out = *a + *b; 
}
```

- 메모리가 겹치면 메모리를 레지스터에 유지할 수 없습니다.
    - 두 번째 줄에서 데이터를 다시 읽어야 합니다.
- `restrict` 키워드는 이 포인터의 수명 동안 다른 포인터가 이 메모리에 접근하지 않을 것임을 보장합니다.
    - 공격적인 성능 최적화가 가능해집니다 - 다시 읽을 필요가 없습니다.
    - 캐시에 거대한 메모리 덩어리를 로드할 수 있습니다.
        - SIMD를 가능하게 합니다.
- restrict 자격을 유지하는 것은 프로그래머의 몫이며, 그렇지 않으면 UB(정의되지 않은 동작)가 발생합니다.

## Static in brackets

이것은 저도 직접 본 적이 없는 기능입니다.

### 문제 - Array decay

```c
void process_data(int data[100]) {
    // ...
}
```

- 표준 C에서 함수 시그니처의 매개변수로 선언된 배열 크기는 중요하지 않으며 단순히 `int *`로 decay(퇴화)됩니다.

```c
void process_data_fast(int data[static 100]) {
    // ...
}
```

- 컴파일러에 전달된 데이터가 절대 `null`이 아니며 최소한 `100` 크기임을 알려줍니다.

```c
// 'data'는 const 포인터이며(다른 곳을 가리키게 할 수 없음),
// restrict(에일리어싱 없음)이며,
// 최소 10개의 요소를 보장합니다!
void ultra_fast_math(int data[const restrict static 10]);
```

- 듣기로는 이게 엄청나게 빠르다고 하네요...?

### Disassemble

```nasm
				lea     rax, [rcx-4]
        mov     r8, rax
        sub     r8, rsi       <-- 포인터 주소 빼기!
        cmp     r8, 8         <-- 8바이트 이하로 겹치는지 확인!
        jbe     .L12          <-- 겹친다면 중단하고 .L12로 점프
        sub     rax, rdx
        cmp     rax, 8
        jbe     .L12          <-- 겹친다면 중단하고 .L12로 점프
```

- fast path - 겹치지 않음
- 겹침을 확인하는 런타임 오버헤드

```nasm
.L12:
        xor     eax, eax
.L9:
        mov     r8d, DWORD PTR [rdx+rax*4]
        add     r8d, DWORD PTR [rsi+rax*4]    <-- 느림, 1개씩 scalar 더하기
        mov     DWORD PTR [rcx+rax*4], r8d
        add     rax, 1
        cmp     rdi, rax
        jne     .L9
```

- slow path - 겹침

[Compiler Explorer - C (x86-64 gcc 15.2)](https://godbolt.org/z/WTjoE3E4T)

- 실제 성능 차이는 미미해 보입니다.
- 다른 경우에는 훨씬 더 빠를 수도 있습니다.

## 기타

- long long int - 64비트 정수
- Variable Length Arrays (VLA)
    - 강등됨
- 한 줄 주소(//) 지원
- snprintf
- `<stdbool.h>` , `<complex.h>`, `<tgmath.h>`, 그리고 `<inttypes.h>`
- IEEE 754-1985
    - 기본적으로 부동 소수점 지원
- Universal character names
    - 사용자 변수에 ascii 이외의 다른 문자를 포함할 수 있도록 허용
