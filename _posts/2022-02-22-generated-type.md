---
title: "Dynamically Generated Type"
category: programming
description: "How to dynamically generate a type at runtime in C#"
---

There are a variety of ways to interact with C#'s type system. You
can:
 * Reflect over existing types
 * Generate types
   * Statically at compile time
     * The normal way
     * With code generation
   * **Dynamically at runtime**

Here's some code that does that last category:

```csharp
namespace Example;

using System.Reflection;
using System.Reflection.Emit;

public interface IFace
{
    string Go();
}

class Program
{
    static IFace Create()
    {
        var assemblyBuilder = AssemblyBuilder.DefineDynamicAssembly(
            new AssemblyName("Generated"),
            AssemblyBuilderAccess.Run
        );
        var moduleBuilder = assemblyBuilder.DefineDynamicModule("Generated");
        var typeBuilder = moduleBuilder.DefineType("GeneratedType", TypeAttributes.Public | TypeAttributes.Class);
        typeBuilder.AddInterfaceImplementation(typeof(IFace));
        var methodBuilder = typeBuilder.DefineMethod("Go", MethodAttributes.Public | MethodAttributes.Virtual, CallingConventions.HasThis, typeof(string), Type.EmptyTypes);
        var generator = methodBuilder.GetILGenerator();
        generator.Emit(OpCodes.Ldstr, "Hello, world!");
        generator.Emit(OpCodes.Ret);

        var type = typeBuilder.CreateType()!;
        var instance = Activator.CreateInstance(type);
        return (IFace)instance!;
    }

    static void Main()
    {
        var instance = Create();
        Console.WriteLine(instance.Go());
    }
}
```

Output:

```
Hello, world!
```