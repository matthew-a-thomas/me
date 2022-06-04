---
title: Loading Loose XAML
category: programming
description: How to parse and load XAML from an embedded resource
---

Suppose you have a custom thingy like this:

```csharp
namespace LooseXaml;

sealed class CustomThingy
{
    public string Property { get; set; } = string.Empty;
}
```

...and you declare it in some XAML:

```xml
<CustomThingy xmlns="clr-namespace:LooseXaml">
    <CustomThingy.Property>Hello, world!</CustomThingy.Property>
</CustomThingy>
```

...and that XAML is in a file named "EmbeddedResource.xaml", which is an
embedded resource:

```
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <OutputType>WinExe</OutputType>
        <TargetFramework>net6.0-windows</TargetFramework>
        <Nullable>enable</Nullable>
        <UseWPF>true</UseWPF>
    </PropertyGroup>

    <ItemGroup>
      <Page Remove="EmbeddedResource.xaml" />
      <EmbeddedResource Include="EmbeddedResource.xaml" />
    </ItemGroup>

</Project>
```

Did you know you can parse and instantiate it?

```csharp
namespace LooseXaml;

using System;
using System.Diagnostics;
using System.IO;
using System.Windows;

public partial class App
{
    protected override void OnStartup(StartupEventArgs e)
    {
        var type = GetType();
        var assembly = type.Assembly;
        using var stream = assembly.GetManifestResourceStream(type, "EmbeddedResource.xaml");
        if (stream is null)
            throw new Exception("Wrong resource name");
        using var reader = new StreamReader(stream);
        var loader = new Loader();
        var customThingy = loader.Load<CustomThingy>(
            reader,
            assembly,
            new [] { assembly }
        );
        if (customThingy is null)
            throw new Exception("Failed to load it. Are you sure that XAML declares a CustomThingy?");
        if (customThingy.Property != "Hello, world!")
            throw new Exception("Loaded it, but its property is wrong for some reason");
        Trace.WriteLine("It works!");
        Environment.Exit(0);
    }
}
```

## The loader

Here's how:

```csharp
namespace LooseXaml;

using System.Collections.Generic;
using System.IO;
using System.Reflection;
using System.Xaml;

public sealed class Loader
{
    public T? Load<T>(
        TextReader textReader,
        Assembly localAssembly,
        IEnumerable<Assembly> assemblies)
        where T : class
    {
        var context = new XamlSchemaContext(assemblies);
        var settings = new XamlXmlReaderSettings
        {
            LocalAssembly = localAssembly,
            ProvideLineInfo = true,
            AllowProtectedMembersOnRoot = true
        };
        using var xamlXmlReader = new XamlXmlReader(
            textReader,
            context,
            settings
        );
        var root = XamlServices.Load(xamlXmlReader);
        return root as T;
    }
}
```

## Things to remember

### XAML only supports parameterless constructors

Well, technically you can use the
[`x:FactoryMethod`](https://docs.microsoft.com/en-us/dotnet/desktop/xaml-services/xfactorymethod-directive)
or
[`x:Arguments`](https://docs.microsoft.com/en-us/dotnet/desktop/xaml-services/xarguments-directive)
directives.

If you do that then remember to include the
`xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"` namespace in your XAML.

### `XamlXmlReaderSettings.LocalAssembly` is important

According to
[Microsoft's documentation](https://docs.microsoft.com/en-us/dotnet/desktop/wpf/advanced/xaml-namespaces-and-namespace-mapping-for-wpf-xaml?view=netframeworkdesktop-4.8#mapping-to-current-assemblies)
you can omit the `;assembly=YourAssembly` part of
`xmlns="clr-namespace:YourNamespace;assembly=YourAssembly"`.

When you do then it'll assume you're talking about the current assembly. Which
is what `XamlXmlReaderSettings.LocalAssembly` points to. So you need to set that
property to the current assembly.

## The point

You can use XAML for more than just WPF!