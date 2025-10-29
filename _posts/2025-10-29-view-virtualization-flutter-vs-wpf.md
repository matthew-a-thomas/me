---
title: View Virtualization in Flutter vs WPF
description: A side-by-side comparison
category: programming
---

I'm learning some Flutter. I'm more familiar with WPF. Here's my experience using view virtualization in both.

## Show a list of one billion numbers

One billion UI elements will easily overwhelm any naive view implementation. You need view virtualization, so that UI elements aren't actually created until they're shown on screen.

It's easy to tell when there is no view virtualization because with one billion items the UI will freeze for longer than anyone is willing to wait.

Once the numbers are shown, I'll grab the scrollbar with the mouse and slide it all over the place to see how responsive it is.

## WPF

```xml
<Window x:Class="WpfApp12.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:WpfApp12"
        xmlns:system="clr-namespace:System;assembly=System.Runtime"
        mc:Ignorable="d"
        Title="MainWindow" Height="450" Width="800"
        d:DataContext="{d:DesignInstance local:MainViewModel}">
    <FrameworkElement.DataContext>
        <local:MainViewModel />
    </FrameworkElement.DataContext>
    <ItemsControl
        ItemsSource="{Binding Numbers, Mode=OneWay}"
        VirtualizingPanel.ScrollUnit="Pixel"
        VirtualizingPanel.IsVirtualizing="True"
        VirtualizingPanel.VirtualizationMode="Recycling">
        <ItemsControl.Template>
            <ControlTemplate>
                <ScrollViewer CanContentScroll="True">
                    <ItemsPresenter />
                </ScrollViewer>
            </ControlTemplate>
        </ItemsControl.Template>
        <ItemsControl.ItemTemplate>
            <DataTemplate DataType="{x:Type system:Int64}">
                <TextBlock Text="{Binding}" />
            </DataTemplate>
        </ItemsControl.ItemTemplate>
        <ItemsControl.ItemsPanel>
            <ItemsPanelTemplate>
                <VirtualizingStackPanel IsItemsHost="True" />
            </ItemsPanelTemplate>
        </ItemsControl.ItemsPanel>
    </ItemsControl>
</Window>
```

```csharp
namespace WpfApp12;

using System.Collections.ObjectModel;
using System.ComponentModel;

public sealed class MainViewModel : INotifyPropertyChanged
{
    public MainViewModel()
    {
        Numbers = new ObservableCollection<long>(Enumerable.Range(0, 1_000_000_000).Select(i => (long)i));
    }

    public event PropertyChangedEventHandler? PropertyChanged
    {
        add { }
        remove { }
    }

    public ObservableCollection<long> Numbers { get; }
}
```

WPF supports view virtualization [out of the box with certain UI elements](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/optimizing-performance-controls#displaying-large-data-sets). But I noticed some jank when I used those, which only got worse the longer I scrolled around. It really felt like GC struggling to keep up. I believe the jank comes from `VirtualizingPanel.VirtualizationMode` defaulting to `Standard` in which "containers are thrown away when offscreen". It was much better when I used `Recycling`. But that required a lot of boilerplate, as seen in the XAML above.

Preview:
![Preview of one billion numbers in a WPF application](/assets/img/view-virtualization-flutter-vs-wpf/wpf-one-billion.png)

**Time to launch: ~10 seconds**. My educated guess is that's about how long it takes to populate a list with one billion items.

**Responsiveness: great**. The scrollbar is only a frame or two behind the mouse cursor.

**Developer experience: meh**. Lots of boilerplate. I had to reach back in memory and read a lot of documentation to re-remember how to do view virtualization, and to learn how to eliminate the jank I mentioned above.

**App size: 180K** plus the one-time cost of the .NET desktop runtime.

## Flutter

```dart
void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  static final numbers = List.generate(1_000_000_000, (index) => index);

  @override
  Widget build(BuildContext context) {
    return Directionality(
      textDirection: TextDirection.ltr,
      child: ListView.builder(
        itemCount: numbers.length,
        prototypeItem: Text(numbers.last.toString()),
        itemBuilder: (context, index) => Text(numbers[index].toString()),
      ),
    );
  }
}
```

Flutter has `ListView.builder`. To get a scrollbar you have to pass in `itemCount` (otherwise it does an infinite scrolling thing).

Without a `prototypeItem` (or `itemExtent` or `itemExtentBuilder`) the UI will lock up when you grab the scrollbar and try to scroll to the bottom of the list. I assume that's because it has to materialize a widget for every single item in order to figure out the height of the _n_ items you're scrolling past.

Preview:
![Preview of one billion numbers in a Flutter application](/assets/img/view-virtualization-flutter-vs-wpf/flutter-one-billion.png)

**Time to launch: ~5 seconds**. Hmm. I guess WPF was doing a lot more work for some reason.

**Responsiveness: great**. The scrollbar is only a frame or two behind the mouse cursor.

**Developer experience: great**.

**App size: 23MB**.