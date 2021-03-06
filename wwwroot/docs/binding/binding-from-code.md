Title: Binding from Code
Order: 60
---

Binding from code in Avalonia works somewhat differently to WPF/UWP. At the low level, Avalonia's
binding system is based on Reactive Extensions' `IObservable` which is then built upon by XAML
bindings (which can also be instantiated in code).

# Subscribing to Changes to a Property

You can subscribe to changes on a property by calling the `GetObservable`
method. This returns an `IObservable<T>` which can be used to listen for changes
to the property:

```csharp
    var textBlock = new TextBlock();
    var text = textBlock.GetObservable(TextBlock.TextProperty);
```

Each property that can be subscribed to has a static readonly field called
`[PropertyName]Property` which is passed to `GetObservable` in order to
subscribe to the property's changes.

`IObservable` (part of Reactive Extensions, or rx for short) is out of scope
for this guide, but here's an example which uses the returned observable to
print a message with the changing property values to the console:

```c#
    var textBlock = new TextBlock();
    var text = textBlock.GetObservable(TextBlock.TextProperty);
    text.Subscribe(value => Console.WriteLine(value + " Changed"));
```

When the returned observable is subscribed, it will return the current value
of the property immediately and then push a new value each time the property
changes. If you don't want the current value, you can use the rx `Skip`
operator:

```c#
    var text = textBlock.GetObservable(TextBlock.TextProperty).Skip(1);
```

# Binding to an observable

You can bind a property to an observable using the `AvaloniaObject.Bind` method:

```csharp
// We use an Rx Subject here so we can push new values using OnNext
var source = new Subject<string>();
var textBlock = new TextBlock();

// Bind TextBlock.Text to source
textBlock.Bind(TextBlock.TextProperty, source);

// Set textBlock.Text to "hello"
source.OnNext("hello");
// Set textBlock.Text to "world!"
source.OnNext("world!");
```

# Setting a binding in an object initializer

It is often useful to set up bindings in object initializers. You can do this using the indexer:

```csharp
var source = new Subject<string>();
var textBlock = new TextBlock
{
    Foreground = Brushes.Red,
    MaxWidth = 200,
    [!TextBlock.TextProperty] = source.ToBinding(),
};
```

Using this method you can also easily bind a property on one control to a property on another:

```csharp
var textBlock1 = new TextBlock();
var textBlock2 = new TextBlock
{
    Foreground = Brushes.Red,
    MaxWidth = 200,
    [!TextBlock.TextProperty] = textBlock1[!TextBlock.TextProperty],
};
```

Of course the indexer can be used outside object initializers too:

```csharp
textBlock2[!TextBlock.TextProperty] = textBlock1[!TextBlock.TextProperty];
```

# Transforming binding values

Because we're working with observables, we can easily transform the values we're binding!

```csharp
var source = new Subject<string>();
var textBlock = new TextBlock
{
    Foreground = Brushes.Red,
    MaxWidth = 200,
    [!TextBlock.TextProperty] = source.Select(x => "Hello " + x).ToBinding(),
};
```

# Using XAML bindings from code

Sometimes when you want the additional features that XAML bindings provide, it's easier to use XAML bindings from code. For example, using only observables you could bind to a property on `DataContext` like this:

```csharp
var textBlock = new TextBlock();
var viewModelProperty = textBlock.GetObservable(TextBlock.DataContext)
    .OfType<MyViewModel>()
    .Select(x => x?.Name);
textBlock.Bind(TextBlock, viewModelProperty);
```

However, it might be preferable to use a XAML binding in this case:

```csharp
var textBlock = new TextBlock
{
    [!TextBlock.TextProperty] = new Binding("Name")
};
```

# Subscribing to a Property on Any Object

The `GetObservable` method returns an observable that tracks changes to a property on a single
instance. However, if you're writing a control you may want to implement an `OnPropertyChanged`
method which isn't tied to an instance of an object.

To do this you can subscribe to 
[`AvaloniaProperty.Changed`](/api/Avalonia/AvaloniaProperty/65237C52) which is an observable which
fires _every time the property is changed on any instance_.

> In WPF this is done by passing a static `PropertyChangedCallback` to the `DependencyProperty`
  registration method, but this only allows the control author to register a property changed
  callback.

In addition there is an `AddClassHandler` extension method which can automatically route the 
event to a method on your control.

For example if you want to listen to changes to your control's `Foo` property
you'd do it like this:

```csharp
static MyControl()
{
    FooProperty.Changed.AddClassHandler<MyControl>(x => x.FooChanged);
}

private void FooChanged(AvaloniaPropertyChangedEventArgs e)
{
    // The 'e' parameter describes what's changed.
}
```
