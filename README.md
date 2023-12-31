最近有一个需求是开启匿名共享

这个可以通过启用guest用户，并创建一个guest用户拥有full control权限的共享来完成
```
net user guest /active:yes
net share sharename2=C:\users\public /gran:guest,FULL /users:100
```

此外还有使用icacls修改共享目录权限，使得guest用户拥有对该目录的完全控制权限

```
icacls "C:\Users\Public" /grant "Guest:(OI)(CI)F"
```

移除权限：

```
icacls "C:\Users\Public" /remove "Guest"
```

这其中主要的障碍就是我们需要更新本地组策略来允许guest用户进行网络认证

![image](https://github.com/wqreytuk/local_policy_editor_study/assets/48377190/a347d5a2-af9f-4ea5-a0ce-2540a7ad4fd2)

可以看到，上面这条组策略默认情况下是不允许guest账户进行remote认证的

在这种情况下，我们使用guest用户连接共享会遇到下面的报错

```
net use \\192.168.13.166\sharename2 "" /user:guest
```

![image](https://github.com/wqreytuk/local_policy_editor_study/assets/48377190/81e848a2-3ae6-40e9-99dc-5980b16e8897)


但是如果我们可以从组策略中移除guest用户，就可以进行匿名认证

为此我进行一些逆向工作，获取到如下的关键调用栈

因为我知道他最后肯定会使用rpc，这个我是通过procmon检测注册表操作获得的情报，lsass.exe会修改security注册表来更改设置（这个注册表键我们无法直接访问）

所以我在`RPCRT4!NdrClientCall3`处下了断点，然后移除用户，触发断点，得到如下调用栈

```
0:023> k
 # Child-SP          RetAddr               Call Site
00 00000000`0387dc68 00007ff9`17cd12df     RPCRT4!NdrClientCall3
01 00000000`0387dc70 00007ff8`9cdcdf06     SCECLI!SceUpdateSecurityProfile+0x11f
02 00000000`0387dd10 00007ff8`9cdcdb38     wsecedit!CEditTemplate::Save+0x23a
03 00000000`0387de00 00007ff8`9cdac121     wsecedit!CEditTemplate::SetDirty+0x34
04 00000000`0387de30 00007ff8`9cdea583     wsecedit!CSnapin::UpdateLocalPolInfo+0x9d
05 00000000`0387dec0 00007ff8`9cdcb20d     wsecedit!CLocalPolRight::SetPrivData+0x33
06 00000000`0387df00 00007ff8`f8476255     wsecedit!CConfigPrivs::OnApply+0x86d
07 00000000`0387dfa0 00007ff8`f8429d2e     MFC42u+0x56255
08 00000000`0387dfd0 00007ff8`f842f9c5     MFC42u+0x9d2e
09 00000000`0387e120 00007ff8`f842bb6f     MFC42u+0xf9c5
0a 00000000`0387e160 00007ff8`f843e209     MFC42u+0xbb6f
0b 00000000`0387e260 00007ff8`9cdfca98     MFC42u+0x1e209
0c 00000000`0387e2a0 00007ff9`1a67e858     wsecedit!AfxWndProcDllStatic+0x48
```


关键调用

```
and     esi, 0FFFFFF8Fh
lea     r9d, [rax+4]    ; rax是上面strcmp的返回值，而进入到这里，rax肯定是0
                        ; 所以这里就是个硬编码的4
mov     r8, [rbx+98h]   ; 所以唯一需要确定的参数只有r8
                        ; rbx+98这个东西我们要回溯一下
                        ;
                        ; rbx应该是一个结构体，0x98偏移可能存储了另外一个结构体的地址
mov     edx, esi        ; 这里esi是固定的，总是8
xor     ecx, ecx        ; 硬编码的0
call    cs:__imp_SceUpdateSecurityProfile
```
__imp_SceUpdateSecurityProfile会通过`SCECLI!SceUpdateSecurityProfile`调用rpc，如果我们能够弄清楚关键的几个参数的含义，就可以自行加载`wsecedit.dll`来进行
函数的调用


[rbx+98]的值如下：

```
0:023> dq /c 1 r8
00000000`23b92e80  fffffffe`0000012e
00000000`23b92e88  fffffffe`fffffffe
00000000`23b92e90  fffffffe`fffffffe
00000000`23b92e98  fffffffe`fffffffe
00000000`23b92ea0  fffffffe`fffffffe
00000000`23b92ea8  fffffffe`fffffffe
00000000`23b92eb0  00000000`00000000
00000000`23b92eb8  00000000`00000000
00000000`23b92ec0  fffffffe`00000000
00000000`23b92ec8  00000000`fffffffe
00000000`23b92ed0  00000000`00000000
00000000`23b92ed8  00000000`23b7ca80
00000000`23b92ee0  00000000`00000000
00000000`23b92ee8  00000000`00000000
00000000`23b92ef0  00000000`00000000
00000000`23b92ef8  00000000`00000000
```

其中我们只需要关注`23b7ca80`这个值，因为其他地方都是固定的

23b7ca80是0x58字段的值，他存储的是一个结构体`_SCE_PRIVILEGE_ASSIGNMENT`

这个结构体并没有公开的文档，但是通过逆向，我大概摸清了他的结构

```
0:023> dq /c 1 23b7ca80
00000000`23b7ca80  00000000`23b7cd50
00000000`23b7ca88  00000000`0000001b
00000000`23b7ca90  00000000`23d159d0
00000000`23b7ca98  00000000`00000000
00000000`23b7caa0  00000000`00000000
00000000`23b7caa8  00000028`4d454d4c
00000000`23b7cab0  00000000`23b7c500
00000000`23b7cab8  00000000`23ba0730
00000000`23b7cac0  fffdfdfd`fffdfdfd
00000000`23b7cac8  9000aafd`94259581
00000000`23b7cad0  006b0063`00610042
00000000`23b7cad8  004f0020`00700075
00000000`23b7cae0  00610072`00650070
00000000`23b7cae8  00730072`006f0074
00000000`23b7caf0  00000000`00000000
00000000`23b7caf8  00000028`4d454d4c
```


我推测，他只有0x18字节，第一个qword是一个字符串
```
00000000`23b7cd50  00650053 00650044 0079006e 0065004e  S.e.D.e.n.y.N.e.
00000000`23b7cd60  00770074 0072006f 004c006b 0067006f  t.w.o.r.k.L.o.g.
00000000`23b7cd70  006e006f 00690052 00680067 00000074  o.n.R.i.g.h.t...
00000000`23b7cd80  4d454d4c 00000030 23b7ca80 00000000  LMEM0......#....
00000000`23b7cd90  23b7caa8 00000000 94f695f4 9000b300  ...#............
00000000`23b7cda0  00610042 006b0063 00700075 004f0020  B.a.c.k.u.p. .O.
00000000`23b7cdb0  00650070 00610072 006f0074 00730072  p.e.r.a.t.o.r.s.
00000000`23b7cdc0  00000000 00000000 4d454d4c 00000028  ........LMEM(...
```

就是这个策略的名称：`SeDenyNetworkLogonRight`

第二个QWORD可能是策略的编号，貌似是固定的，这个明天去公司电脑上验证一下（不同的策略设置拥有不同的编号）

第三个QWORD是用户名字符串的**地址的地址**

```
0:023> dc poi(23d159d0)
00000000`23b63840  00450044 004b0053 004f0054 002d0050  D.E.S.K.T.O.P.-.
00000000`23b63850  00500037 00370045 004e0050 005c0034  7.P.E.7.P.N.4.\.
00000000`23b63860  00750047 00730065 00000074 00000000  G.u.e.s.t.......
00000000`23b63870  00000000 00000000 97168aaa 80001800  ................
```

如果是进行移除操作的话，直接将第三个qword值置为0即可



# 进一步测试

进一步测试，大致总结出了+58字段结构体的结构
```
typedef strcut _0x58 {
    QWORD _unicode_addr;
    QWORD _sui_bian;
    QWORD _addr_of_unicode_addr;
    QWORD 0;
}
```


参数传入情况
```
1p 0
2p 8
3p 未知结构体，内嵌_0x58结构体地址
4p 4
```

添加时第三个参数的情况

```
00000000`22a18d50  fffffffe`0000012e
00000000`22a18d58  fffffffe`fffffffe
00000000`22a18d60  fffffffe`fffffffe
00000000`22a18d68  fffffffe`fffffffe
00000000`22a18d70  fffffffe`fffffffe
00000000`22a18d78  00000000`fffffffe
00000000`22a18d80  00000000`00000000
00000000`22a18d88  00000000`00000000
00000000`22a18d90  fffffffe`00000000
00000000`22a18d98  00000000`fffffffe
00000000`22a18da0  00000000`00000000
00000000`22a18da8  00000000`22a27480
00000000`22a18db0  00000000`00000000
00000000`22a18db8  00000000`00000000
00000000`22a18dc0  00000000`00000000
00000000`22a18dc8  00000000`00000000
```

删除时第三个参数的情况

```
0:002> dq /c 1 0000000022a18d50  
00000000`22a18d50  00000000`0000012e
00000000`22a18d58  00000000`00000000
00000000`22a18d60  00000000`00000000
00000000`22a18d68  00000000`00000000
00000000`22a18d70  00000000`00000000
00000000`22a18d78  00000000`00000000
00000000`22a18d80  00000000`00000000
00000000`22a18d88  00000000`00000000
00000000`22a18d90  00000000`00000000
00000000`22a18d98  00000000`00000000
00000000`22a18da0  00000000`00000000
00000000`22a18da8  00000000`22a27480
00000000`22a18db0  00000000`00000000
00000000`22a18db8  00000000`00000000
00000000`22a18dc0  00000000`00000000
00000000`22a18dc8  00000000`00000000

```


# 工具

https://github.com/wqreytuk/for_public_local_policy_editor_study/blob/main/%E5%85%B3%E6%B3%A8%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%8F%B7%E3%80%8A%E6%88%91%E5%90%83%E4%BD%A0%E5%AE%B6%E7%B1%B3%E4%BA%86%E3%80%8B%E5%9B%9E%E5%A4%8Dbqg%E8%8E%B7%E5%8F%96%E5%AF%86%E7%A0%81.7z

# windows7和windows10测试通过

实际测试发现，只要是administrator权限就可以了

开启后直接使用如下方式链接即可

```
net use \\targetIP\sharename "" /user:Guest
```
