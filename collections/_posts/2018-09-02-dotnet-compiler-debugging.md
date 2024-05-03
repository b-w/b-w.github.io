---
layout: post
title: "Static variables, constructors, and JIT optimizations"
categories: blog
---

We ran into an interesting problem at work the other day. A unit test was always passing locally, but failing on the build server. Here's a minimal reproduction of the code involved, so we can take a look at what went wrong.

The (relevant bits of the) class that was being tested looked something like this:

```csharp
public class Foo
{
    private static readonly ConcurrentDictionary<int, Foo> _map
        = new ConcurrentDictionary<int, Foo>();

    public Foo(int id)
    {
        Trace.WriteLine($"Constructed Foo, Id = {id}");

        if (!_map.TryAdd(id, this))
        {
            throw new ArgumentException($"Id {id} is already registered.");
        }
    }
}
```

It kept a static dictionary of constructed instances of itself, and did not allow multiple instances with the same Id to be constructed. Here's the corresponding unit test:

```csharp
[TestClass]
public class FooTests
{
    [TestMethod, ExpectedException(typeof(ArgumentException))]
    public void Ctor_Given_Id_Already_In_Use_Should_Throw()
    {
        // act
        var foo = new FooFake(1);
    }

    public class FooFake : Foo
    {
        static readonly Foo c1 = new FooFake(1);
        static readonly Foo c2 = new FooFake(2);
        static readonly Foo c3 = new FooFake(3);

        public FooFake(int id)
            : base(id)
        {
        }
    }
}
```

I'll admit that in isolation this looks like an odd way of testing this particular function, but that's beside the point here. This test ran fine locally, but kept failing on the build server. We quickly realized the problem was not with the build server itself, but with the build configuration that was being used. We ran the test locally using debug builds, while the build server was using a release build. When we tried a release build locally, we were able to reproduce the test failing.

Looking at the trace output produced during the test, we could see the debug build outputting the following:

```
Debug Trace:
Constructed Foo, Id = 1
Constructed Foo, Id = 2
Constructed Foo, Id = 3
Constructed Foo, Id = 1
```

...while running the test in release mode produced this:

```csharp
Debug Trace:
Constructed Foo, Id = 1
```

Based on this, we concluded that the unused c1, c2, and c3 variables were being optimized away in the release build. Knowing the cause, we were able to fix our test quite easily. However, out of curiosity I decided to dig a little deeper, to see what was going on under the hood. To test our hunch about compiler optimizations being the culprit, I opened the debug and release builds in dnSpy to compare their IL.

Here's the FooFake class, debug build:

```csharp
// Token: 0x02000003 RID: 3
.class nested public auto ansi beforefieldinit FooFake
    extends [ConsoleApp]ConsoleApp.Foo
{
    // Fields
    // Token: 0x04000001 RID: 1
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c1
    // Token: 0x04000002 RID: 2
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c2
    // Token: 0x04000003 RID: 3
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c3

    // Methods
    // Token: 0x06000003 RID: 3 RVA: 0x0000206E File Offset: 0x0000026E
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 10 (0xA) bytes
        .maxstack 8

        /* 0x0000026F 02           */ IL_0000: ldarg.0
        /* 0x00000270 03           */ IL_0001: ldarg.1
        /* 0x00000271 281300000A   */ IL_0002: call      instance void [ConsoleApp]ConsoleApp.Foo::.ctor(int32)
        /* 0x00000276 00           */ IL_0007: nop
        /* 0x00000277 00           */ IL_0008: nop
        /* 0x00000278 2A           */ IL_0009: ret
    } // end of method FooFake::.ctor

    // Token: 0x06000004 RID: 4 RVA: 0x00002079 File Offset: 0x00000279
    .method private hidebysig specialname rtspecialname static 
        void .cctor () cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 34 (0x22) bytes
        .maxstack 8

        /* 0x0000027A 17           */ IL_0000: ldc.i4.1
        /* 0x0000027B 7303000006   */ IL_0001: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000280 8001000004   */ IL_0006: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c1
        /* 0x00000285 18           */ IL_000B: ldc.i4.2
        /* 0x00000286 7303000006   */ IL_000C: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x0000028B 8002000004   */ IL_0011: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c2
        /* 0x00000290 19           */ IL_0016: ldc.i4.3
        /* 0x00000291 7303000006   */ IL_0017: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000296 8003000004   */ IL_001C: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c3
        /* 0x0000029B 2A           */ IL_0021: ret
    } // end of method FooFake::.cctor

} // end of class FooFake
```

...and here's the release build:

```csharp
// Token: 0x02000003 RID: 3
.class nested public auto ansi beforefieldinit FooFake
    extends [ConsoleApp]ConsoleApp.Foo
{
    // Fields
    // Token: 0x04000001 RID: 1
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c1
    // Token: 0x04000002 RID: 2
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c2
    // Token: 0x04000003 RID: 3
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c3

    // Methods
    // Token: 0x06000003 RID: 3 RVA: 0x00002061 File Offset: 0x00000261
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 8 (0x8) bytes
        .maxstack 8

        /* 0x00000262 02           */ IL_0000: ldarg.0
        /* 0x00000263 03           */ IL_0001: ldarg.1
        /* 0x00000264 281300000A   */ IL_0002: call      instance void [ConsoleApp]ConsoleApp.Foo::.ctor(int32)
        /* 0x00000269 2A           */ IL_0007: ret
    } // end of method FooFake::.ctor

    // Token: 0x06000004 RID: 4 RVA: 0x0000206A File Offset: 0x0000026A
    .method private hidebysig specialname rtspecialname static 
        void .cctor () cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 34 (0x22) bytes
        .maxstack 8

        /* 0x0000026B 17           */ IL_0000: ldc.i4.1
        /* 0x0000026C 7303000006   */ IL_0001: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000271 8001000004   */ IL_0006: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c1
        /* 0x00000276 18           */ IL_000B: ldc.i4.2
        /* 0x00000277 7303000006   */ IL_000C: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x0000027C 8002000004   */ IL_0011: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c2
        /* 0x00000281 19           */ IL_0016: ldc.i4.3
        /* 0x00000282 7303000006   */ IL_0017: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000287 8003000004   */ IL_001C: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c3
        /* 0x0000028C 2A           */ IL_0021: ret
    } // end of method FooFake::.cctor

} // end of class FooFake
```

Besides the release build removing a few nop instructions in the constructor, they are identical.

So, it's not a compiler issue. That just leaves the runtime itself as a suspect. How about the JIT, then? We'll place the following breakpoints in our class:

![](/assets/img/blog/2018/09/vs-test-debugging.png)

When debugging the debug build, both breakpoints are hit. We can then right-click the code and select "Go To Disassembly" to see what the JIT compiler has produced. In this case, we can clearly see that the static constructor that initializes the three Foo instances has been compiled to native code:

```
--- FooTests.cs ----------------------------------------------------------------
            static readonly Foo c1 = new FooFake(1);
06EB3A20  push        ebp  
06EB3A21  mov         ebp,esp  
06EB3A23  push        edi  
06EB3A24  push        esi  
06EB3A25  push        ebx  
06EB3A26  sub         esp,38h  
06EB3A29  xor         eax,eax  
06EB3A2B  mov         dword ptr [ebp-3Ch],eax  
06EB3A2E  mov         dword ptr [ebp-40h],eax  
06EB3A31  mov         dword ptr [ebp-44h],eax  
06EB3A34  mov         dword ptr [ebp-10h],eax  
06EB3A37  mov         dword ptr [ebp-1Ch],eax  
06EB3A3A  cmp         dword ptr ds:[6C3A298h],0  
06EB3A41  je          06EB3A48  
06EB3A43  call        627E4030  
06EB3A48  mov         ecx,6C9B2C4h  
06EB3A4D  call        00C430F4  
06EB3A52  mov         dword ptr [ebp-3Ch],eax  
06EB3A55  mov         ecx,dword ptr [ebp-3Ch]  
06EB3A58  mov         edx,1  
06EB3A5D  call        06BC43C8  
06EB3A62  mov         eax,dword ptr [ebp-3Ch]  
06EB3A65  lea         edx,ds:[3B7EF88h]  
06EB3A6B  call        6249E6C0  
            static readonly Foo c2 = new FooFake(2);
06EB3A70  mov         ecx,6C9B2C4h  
06EB3A75  call        00C430F4  
06EB3A7A  mov         dword ptr [ebp-40h],eax  
06EB3A7D  mov         ecx,dword ptr [ebp-40h]  
06EB3A80  mov         edx,2  
06EB3A85  call        06BC43C8  
06EB3A8A  mov         eax,dword ptr [ebp-40h]  
06EB3A8D  lea         edx,ds:[3B7EF8Ch]  
06EB3A93  call        6249E6C0  
            static readonly Foo c3 = new FooFake(3);
06EB3A98  mov         ecx,6C9B2C4h  
06EB3A9D  call        00C430F4  
06EB3AA2  mov         dword ptr [ebp-44h],eax  
06EB3AA5  mov         ecx,dword ptr [ebp-44h]  
06EB3AA8  mov         edx,3  
06EB3AAD  call        06BC43C8  
06EB3AB2  mov         eax,dword ptr [ebp-44h]  
06EB3AB5  lea         edx,ds:[3B7EF90h]  
06EB3ABB  call        6249E6C0  
06EB3AC0  nop  
06EB3AC1  lea         esp,[ebp-0Ch]  
06EB3AC4  pop         ebx  
06EB3AC5  pop         esi  
06EB3AC6  pop         edi  
06EB3AC7  pop         ebp  
06EB3AC8  ret  
--- No source file -------------------------------------------------------------
06EB3AC9  add         byte ptr [eax],al  
06EB3ACB  add         byte ptr [eax],al  
06EB3ACD  add         byte ptr [eax],al  
06EB3ACF  add         ah,bh  
06EB3AD1  jge         06EB3ABD  
06EB3AD3  push        es  
06EB3AD4  add         byte ptr [eax],al  
06EB3AD6  add         byte ptr [eax],al  
06EB3AD8  les         edi,fword ptr [ebp-16h]  
06EB3ADB  push        es  
06EB3ADC  test        al,0B2h  
06EB3ADE  leave  
06EB3ADF  push        es  
--- FooTests.cs ----------------------------------------------------------------
                : base(id)
06EB3AE0  push        ebp  
06EB3AE1  mov         ebp,esp  
06EB3AE3  push        edi  
06EB3AE4  push        esi  
                : base(id)
06EB3AE5  push        ebx  
06EB3AE6  sub         esp,44h  
06EB3AE9  mov         esi,ecx  
06EB3AEB  lea         edi,[ebp-50h]  
06EB3AEE  mov         ecx,11h  
06EB3AF3  xor         eax,eax  
06EB3AF5  rep stos    dword ptr es:[edi]  
06EB3AF7  mov         ecx,esi  
06EB3AF9  xor         eax,eax  
06EB3AFB  mov         dword ptr [ebp-1Ch],eax  
06EB3AFE  mov         dword ptr [ebp-3Ch],ecx  
06EB3B01  mov         dword ptr [ebp-40h],edx  
06EB3B04  cmp         dword ptr ds:[6C3A298h],0  
06EB3B0B  je          06EB3B12  
06EB3B0D  call        627E4030  
06EB3B12  mov         ecx,dword ptr [ebp-3Ch]  
06EB3B15  mov         edx,dword ptr [ebp-40h]  
06EB3B18  call        06BC43B0  
06EB3B1D  nop  
            {
06EB3B1E  nop  
                Trace.WriteLine($"FooFake {id}");
06EB3B1F  mov         ecx,614AF2DCh  
06EB3B24  call        00C430F4  
06EB3B29  mov         dword ptr [ebp-44h],eax  
06EB3B2C  mov         eax,dword ptr ds:[3B840FCh]  
06EB3B32  mov         dword ptr [ebp-4Ch],eax  
06EB3B35  mov         eax,dword ptr [ebp-44h]  
06EB3B38  mov         edx,dword ptr [ebp-40h]  
06EB3B3B  mov         dword ptr [eax+4],edx  
06EB3B3E  mov         eax,dword ptr [ebp-44h]  
06EB3B41  mov         dword ptr [ebp-50h],eax  
06EB3B44  mov         ecx,dword ptr [ebp-4Ch]  
06EB3B47  mov         edx,dword ptr [ebp-50h]  
06EB3B4A  call        613AE100  
06EB3B4F  mov         dword ptr [ebp-48h],eax  
06EB3B52  mov         ecx,dword ptr [ebp-48h]  
06EB3B55  call        dword ptr ds:[6D63BFCh]  
06EB3B5B  nop  
            }
06EB3B5C  nop  
06EB3B5D  lea         esp,[ebp-0Ch]  
06EB3B60  pop         ebx  
06EB3B61  pop         esi  
06EB3B62  pop         edi  
06EB3B63  pop         ebp  
06EB3B64  ret
```

When repeating these steps in release mode, things are a little different. To be able to debug in release mode in the first place, we'll need to make sure the following preconditions are met:

1.  In our project's build settings, debugging information has been set to "pdb-only".
2.  In the debugging options, "Enable Just My Code" and "Suppress JIT optimization on module load" are unchecked.

When we now run the test, we find that the first breakpoint is not hit at all; only the second breakpoint is hit. When we try to "Go To Disassembly" on the c1, c2, or c3 lines, Visual Studio tells us that "Disassembly cannot be displayed for the source location. The expression has not yet been translated to native machine code."

This is, of course, because those particular lines have been optimized away. If we look at the disassembly for the FooFake constructor, we see that it is still there:

```
--- FooTests.cs ----------------------------------------------------------------
                : base(id)
06B439B0  push        ebp  
06B439B1  mov         ebp,esp  
06B439B3  push        edi  
06B439B4  push        esi  
06B439B5  sub         esp,10h  
06B439B8  xor         eax,eax  
06B439BA  mov         dword ptr [ebp-18h],eax  
06B439BD  mov         dword ptr [ebp-14h],eax  
06B439C0  mov         dword ptr [ebp-10h],eax  
06B439C3  mov         dword ptr [ebp-0Ch],eax  
06B439C6  mov         esi,edx  
06B439C8  mov         edx,esi  
06B439CA  call        dword ptr ds:[692B204h]  
            {
                Trace.WriteLine($"FooFake {id}");
06B439D0  mov         ecx,614AF2DCh  
06B439D5  call        009A30F4  
06B439DA  mov         dword ptr [eax+4],esi  
06B439DD  mov         edx,eax  
06B439DF  lea         edi,[ebp-18h]  
06B439E2  xorps       xmm0,xmm0  
06B439E5  movq        mmword ptr [edi],xmm0  
06B439E9  movq        mmword ptr [edi+8],xmm0  
06B439EE  lea         ecx,[ebp-18h]  
06B439F1  call        61BA0F50  
06B439F6  lea         eax,[ebp-18h]  
06B439F9  push        dword ptr [eax+0Ch]  
06B439FC  push        dword ptr [eax+8]  
06B439FF  push        dword ptr [eax+4]  
06B43A02  push        dword ptr [eax]  
06B43A04  mov         edx,dword ptr ds:[38C40FCh]  
06B43A0A  xor         ecx,ecx  
06B43A0C  call        61397A00  
06B43A11  mov         ecx,eax  
06B43A13  call        dword ptr ds:[69F40E0h]  
06B43A19  lea         esp,[ebp-8]  
06B43A1C  pop         esi  
06B43A1D  pop         edi  
06B43A1E  pop         ebp  
06B43A1F  ret
```

However, the static constructor is nowhere to be found. The most likely explanation is that the JIT, as part of the various optimizations it performs in release configuration, has detected that the c1, c2, and c3 variables aren't used anywhere in the program, and has decided to remove them.

Okay, this all makes sense so far. But here's where things get a little funny. Because what happens if we change our class to the following:

```csharp
public class FooFake : Foo
{
    static readonly Foo c1;
    static readonly Foo c2;
    static readonly Foo c3;

    static FooFake()
    {
        c1 = new FooFake(1);
        c2 = new FooFake(2);
        c3 = new FooFake(3);
    }

    public FooFake(int id)
        : base(id)
    {
    }
}
```

We now explicitly initialize the variables from the static constructor.

And again, here's the IL that gets produced in debug mode:

```csharp
// Token: 0x02000003 RID: 3
.class nested public auto ansi FooFake
    extends [ConsoleApp]ConsoleApp.Foo
{
    // Fields
    // Token: 0x04000001 RID: 1
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c1
    // Token: 0x04000002 RID: 2
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c2
    // Token: 0x04000003 RID: 3
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c3

    // Methods
    // Token: 0x06000003 RID: 3 RVA: 0x0000206E File Offset: 0x0000026E
    .method private hidebysig specialname rtspecialname static 
        void .cctor () cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 35 (0x23) bytes
        .maxstack 8

        /* 0x0000026F 00           */ IL_0000: nop
        /* 0x00000270 17           */ IL_0001: ldc.i4.1
        /* 0x00000271 7304000006   */ IL_0002: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000276 8001000004   */ IL_0007: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c1
        /* 0x0000027B 18           */ IL_000C: ldc.i4.2
        /* 0x0000027C 7304000006   */ IL_000D: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000281 8002000004   */ IL_0012: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c2
        /* 0x00000286 19           */ IL_0017: ldc.i4.3
        /* 0x00000287 7304000006   */ IL_0018: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x0000028C 8003000004   */ IL_001D: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c3
        /* 0x00000291 2A           */ IL_0022: ret
    } // end of method FooFake::.cctor

    // Token: 0x06000004 RID: 4 RVA: 0x00002092 File Offset: 0x00000292
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 10 (0xA) bytes
        .maxstack 8

        /* 0x00000293 02           */ IL_0000: ldarg.0
        /* 0x00000294 03           */ IL_0001: ldarg.1
        /* 0x00000295 281300000A   */ IL_0002: call      instance void [ConsoleApp]ConsoleApp.Foo::.ctor(int32)
        /* 0x0000029A 00           */ IL_0007: nop
        /* 0x0000029B 00           */ IL_0008: nop
        /* 0x0000029C 2A           */ IL_0009: ret
    } // end of method FooFake::.ctor

} // end of class FooFake
```

...and release mode:

```csharp
// Token: 0x02000003 RID: 3
.class nested public auto ansi FooFake
    extends [ConsoleApp]ConsoleApp.Foo
{
    // Fields
    // Token: 0x04000001 RID: 1
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c1
    // Token: 0x04000002 RID: 2
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c2
    // Token: 0x04000003 RID: 3
    .field private static initonly class [ConsoleApp]ConsoleApp.Foo c3

    // Methods
    // Token: 0x06000003 RID: 3 RVA: 0x00002061 File Offset: 0x00000261
    .method private hidebysig specialname rtspecialname static 
        void .cctor () cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 34 (0x22) bytes
        .maxstack 8

        /* 0x00000262 17           */ IL_0000: ldc.i4.1
        /* 0x00000263 7304000006   */ IL_0001: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000268 8001000004   */ IL_0006: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c1
        /* 0x0000026D 18           */ IL_000B: ldc.i4.2
        /* 0x0000026E 7304000006   */ IL_000C: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x00000273 8002000004   */ IL_0011: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c2
        /* 0x00000278 19           */ IL_0016: ldc.i4.3
        /* 0x00000279 7304000006   */ IL_0017: newobj    instance void TestApp.FooTests/FooFake::.ctor(int32)
        /* 0x0000027E 8003000004   */ IL_001C: stsfld    class [ConsoleApp]ConsoleApp.Foo TestApp.FooTests/FooFake::c3
        /* 0x00000283 2A           */ IL_0021: ret
    } // end of method FooFake::.cctor

    // Token: 0x06000004 RID: 4 RVA: 0x00002084 File Offset: 0x00000284
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Header Size: 1 byte
        // Code Size: 8 (0x8) bytes
        .maxstack 8

        /* 0x00000285 02           */ IL_0000: ldarg.0
        /* 0x00000286 03           */ IL_0001: ldarg.1
        /* 0x00000287 281300000A   */ IL_0002: call      instance void [ConsoleApp]ConsoleApp.Foo::.ctor(int32)
        /* 0x0000028C 2A           */ IL_0007: ret
    } // end of method FooFake::.ctor

} // end of class FooFake
```

Aside from the constructors being in a different order for some reason, they seem to be pretty much the same. So this shouldn't make a difference, right?

Except that it does, and our test is now also green in release mode. If we place a breakpoint in the static constructor and debug the release build, it will get hit. How about that, huh?

I actually don't have a proper explanation for this (I simply don't know enough about the .NET compiler internals to speak with any authority on it), but I can take a guess. The only relevant difference between the first versions of the IL and the second set, is the absence of the beforefieldinit keyword in the latter. I know that this keyword influences when the type initializer gets called, and that having it present means the initializer gets called before the first reference to a static field. Since there are no references to the c1, c2, and c3 fields, the type initializer won't ever have to be called, and can be optimized away. I think. In the latter case, having the beforefieldinit modifier not present means the type initializer gets called before the first reference to any static or instance field or before the first call to any static, virtual, or instance method of this type. Since we do have one of those (assuming the constructor counts), the type initializer would have to be called, and can't be optimized away. I think. Again.

So even though I previously thought that the first and second versions of the FooFake class would be identical, it turns out they aren't. It would explain why our unit test fails in release mode, but it doesn't explain why the test passes in debug mode. If I understand things correctly (and I'm not sure that I do), the beforefieldinit modifier only guarantees that the type initializer gets called before the first reference to any static field, but other than that it's completely nondeterministic. So the fact that it gets called before the unit test calls the constructor is just lucky, I guess.

This has been an interesting deep dive, but I'll have to leave it at this for now. I'll have to read up on this subject some more.
