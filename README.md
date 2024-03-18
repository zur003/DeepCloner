This repository is a fork of [DeepCloner.Core](https://github.com/adimosh/DeepCloner) which is in turn a fork of 
[DeepCloner](https://github.com/force-net/DeepCloner/). It differs from DeepCloner.Core only in that fields tagged with 
the "NonSerialized" attribute are not cloned, but are initialized to the default value for their type (usually either 0 
or null). This modification was made because a common technique for performing deep clones in .Net has been to use 
BinaryFormatter to serialize and deserialize an object. However, Microsoft has deprecated the use of BinaryFormatter, 
and is expected to remove it completely in the release of .NET 9 at the end of 2024. This fork was developed specifically 
for use by the [ApsimInitiative](https://github.com/APSIMInitiative), which historically has employed BinaryFormetter for 
object cloning, but with reliance on the use of "NonSerialized" tags to prevent specific sub-fields from being included 
in the cloning process.


# DeepCloner.Core

Library with extenstion to clone objects for .NET. It can deep or shallow copy objects. In deep cloning all object graph is maintained. Library actively uses code-generation in runtime as result object cloning is blazingly fast.
Also, there are some performance tricks to increase cloning speed (see tests below).
Objects are copied by its' internal structure, **no** methods or constructuctors are called for cloning objects. As result, you can copy **any** object, but we don't recommend to copy objects which are binded to native resources or pointers. It can cause unpredictable results (but object will be cloned).

You don't need to mark objects somehow, like Serializable-attribute, or restrict to specific interface. Absolutely any object can be cloned by this library. And this object doesn't have any ability to determine that he is clone (except with very specific methods).

Also, there is no requirement to specify object type for cloning. Object can be casted to inteface or as an abstract object, you can clone array of ints as abstract Array or IEnumerable, even null can be cloned without any errors.

Installation through Nuget:

```powershell
Install-Package DeepCloner.Core
```

This library is a fork of [DeepCloner](https://github.com/force-net/DeepCloner/) that has been made for the purpose of modernizing DeepCloner and bringing it in line with the rest of the world (.NET Standard 1.3? .NET 4.0? Who can't afford to upgrade from those to a supported standard, really?). It is needed in at least one of my commercial projects.

## Supported Frameworks

DeepCloner.Core works for .NET 4.6.2 or higher or for .NET 6.

## Usage

Deep cloning any object:

```csharp
var clone = new { Id = 1, Name = "222" }.DeepClone();
```

With a reference to same object:

```csharp
// public class Tree { public Tree ParentTree; }
var t = new Tree();
t.ParentTree = t;
var cloned = t.DeepClone();
Console.WriteLine(cloned.ParentTree == cloned); // True
```

Or as object:

```csharp
var date = DateTime.Now;
var obj = (object)date;
obj.DeepClone().GetType(); // DateTime
```

Shallow cloning (clone only same object, not objects that object relate to) 

```csharp
var clone = new { Id = 1, Name = "222" }.ShallowClone();
```

Cloning to existing object (can be useful for _copying_ constructors, creating wrappers or for keeping references to same object)

```csharp
public class Derived : BaseClass
{
    public Derived(BaseClass parent)
    {
        parent.DeepCloneTo(this); // now this has every field from parent
    }
}
```

Please, note, that _DeepCloneTo_ and _ShallowCloneTo_ requre that object should be class (it is useless for structures) and derived class must be real descendant of parent class (or same type). In another words, this code will not work:

```csharp
public class Base {}
public class Derived1 : Base {}
public class Derived2 : Base {}

var b = (Base)new Derived1(); // casting derived to parent
var derived2 = new Derived2();
// will compile, but will throw an exception in runtime, Derived1 is not parent for Derived2
b.DeepCloneTo(derived2); 
```

## Details

You can use deep clone of objects for a lot of situations, e.g.:
* Emulation of external service or _deserialization elimination_ (e.g. in Unit Testing). When code has received object from external source, code can change it (because object for code is *own*).
* ReadOnly object replace. Instead of wrapping your object to readonly object, you can clone object and target code can do anything with it without any restriction.
* Caching. You can cache data locally and want to ensurce that cached object hadn't been changed by other code
 
You can use shallow clone as fast, light version of deep clone (if your situation allows that). Main difference between deep and shallow clone in code below:

```csharp
// public class A { public B B; }
// public class B { public int X; }
var b = new B { X = 1 };
var a = new A { B = b };
var deepClone = a.DeepClone();
deepClone.B.X = 2;
Console.WriteLine(a.B.X); // 1
var shallowClone = a.ShallowClone();
shallowClone.B.X = 2;
Console.WriteLine(a.B.X); // 2
```

So, deep cloning is guarantee that all changes of cloned object does not affect original. Shallow clone does not guarantee this. But it faster, because deep clone of object can copy big graph of related objects and related objects of related objects and related related related objects, and... so on...

This library does not call any method of cloning object: constructors, Equals, GetHashCode, propertes - nothing is called. So, it is impossible for cloning object to receive information about cloning, throw an exception or return invalid data. 
If you need to call some methods after cloning, you can wrap cloning call to another method which will perform required actions.

Extension methods in library are generic, but it is not require to specifify type for cloning. You can cast your objects to System.Object, or to an interface, add fields will be carefully copied to new object.

## Performance tricks

We perform a lot of performance tricks to ensure cloning is really fast. Here is some of them:

* Using a shallow cloning instead of deep cloning if object is safe for this operation
* Copying an whole object and updating only required fields
* Special handling for structs (can be copied without any cloning code, if possible)
* Cloners caching
* Optimizations for copying simple objects (reduced number of checks to ensure good performance)
* Special handling of reference count for simple objects, that is faster than default dictionary
* Constructors analyzing to select best variant of object construction
* Direct copying of arrays if possible
* Custom handling of one-dimensional and two-dimensional zero-based arrays (most of arrays in usual code)

## License

[MIT](https://github.com/adimosh/DeepCloner/blob/develop/LICENSE) license - original fork license also [MIT](https://github.com/force-net/DeepCloner/blob/develop/LICENSE) license.