Equality in C# and .Net

If you think that comparing two objects for equality is trivial, then you are probably a junior .Net programmer. This short article will introduce you to every mechanism in .Net that is used to determine if two values are equal.

# Part 1. Reference types

Let's start with the easiest situation first. How do two instances of `object` or two instances of a completely regular descendant from `object` determine their equality? We'll use the following two variables throughout the examples:

```c#
var objA = new SampleClass();
var objB = new SampleClass();
```

## Operator `==`

```c#
var op_Equality_class = objA == objB;
```

We haven't defined operator `==` for our `SampleClass`. However, since it's a reference type, it is automatically translated into a `ceq` CIL instruction. TODO: add from CIL manual.

## `static ReferenceEquals`

```c#
var objectReferenceEquals_class = object.ReferenceEquals(objA, objB);
```

`ReferenceEquals` is a very trivial static method:

```c#
public static bool ReferenceEquals(Object objA, Object objB)
{
    return objA == objB;
}
```

## `virtual Equals`

```c#
var thisEquals_class = objA.Equals(objB);
```

The only virtual method of the four, the base `Equals(object)` is implemented using a runtime helper:

```c#
public virtual bool Equals(Object obj)
{
    return RuntimeHelpers.Equals(this, obj);
}
```

The helper method is implemented in unmanaged code

```c#
[System.Security.SecuritySafeCritical]  // auto-generated
[ResourceExposure(ResourceScope.None)]
[MethodImplAttribute(MethodImplOptions.InternalCall)]
public new static extern bool Equals(Object o1, Object o2);
```

The unmanaged code itself is in `src/classlibnative/bcltype/objectnative.cpp`:

```c++
FCIMPL2(FC_BOOL_RET, ObjectNative::Equals, Object *pThisRef, Object *pCompareRef)
{
    CONTRACTL
    {
        FCALL_CHECK;
        INJECT_FAULT(FCThrow(kOutOfMemoryException););
    }
    CONTRACTL_END;
    
    if (pThisRef == pCompareRef)    
        FC_RETURN_BOOL(TRUE);

    // Since we are in FCALL, we must handle NULL specially.
    if (pThisRef == NULL || pCompareRef == NULL)
        FC_RETURN_BOOL(FALSE);

    MethodTable *pThisMT = pThisRef->GetMethodTable();

    // If it's not a value class, don't compare by value
    if (!pThisMT->IsValueType())
        FC_RETURN_BOOL(FALSE);

    // Make sure they are the same type.
    if (pThisMT != pCompareRef->GetMethodTable())
        FC_RETURN_BOOL(FALSE);

    // Compare the contents (size - vtable - sync block index).
    DWORD dwBaseSize = pThisRef->GetMethodTable()->GetBaseSize();
    if(pThisRef->GetMethodTable() == g_pStringClass)
        dwBaseSize -= sizeof(WCHAR);
    BOOL ret = memcmp(
        (void *) (pThisRef+1), 
        (void *) (pCompareRef+1), 
        dwBaseSize - sizeof(Object) - sizeof(int)) == 0;

    FC_GC_POLL_RET();

    FC_RETURN_BOOL(ret);
}
FCIMPLEND
```

As you can see from the code, the unmanaged method works for both reference and value types. This is unusual, as value types have their own virtual implementation.

## `static Equals`

```c#
var objectEquals_class = object.Equals(objA, objB);
```

The implementation of this method is rather straightforward:

```c#
public static bool Equals(Object objA, Object objB) 
{
    if (objA==objB) {
        return true;
    }
    if (objA==null || objB==null) {
        return false;
    }
    return objA.Equals(objB);
}
```

As you can see, it does some null checks and fall back to the virtual method. This, of course, means that it does the null checks twice if you do not override it.

## Conclusion

All four methods compare reference types using reference equality (i.e., object identity). Both static and virtual `Equals` methods can change their behavior in child classes and will continue to work if you upcast your values to `object`, while `ReferenceEquals` will always use reference equality.

Operator `==` can be reimplemented in a child class, but if the value on either side of it is upcast to `object`, the version from `object` will be used, again utilizing reference equality.

# Part 2. Value types

Value types are technically descendants of `System.ValueType` which derives from `System.Object`, but they are a very special case. They have no object identity, so they are usually compared by value, not by reference. This means that using `ReferenceEquals` on value types is a very bad idea: it will always return false. 

## Operator `==`

There's no predefined operator `==` for all value types like there's one for reference types and C# will actually not allow you to use it on value types that do not define it themselves.

## `virtual Equals`

`Equals` is overridden in `System.ValueType`, so you can use it to compare two custom structs either via the static or the virtual method, but one look at the source will make it clear why this is a wrong idea:

```c#
[System.Security.SecuritySafeCritical]
public override bool Equals (Object obj) {
    BCLDebug.Perf(false, "ValueType::Equals is not fast.  "+this.GetType().FullName+" should override Equals(Object)");
    if (null==obj) {
        return false;
    }
    RuntimeType thisType = (RuntimeType)this.GetType();
    RuntimeType thatType = (RuntimeType)obj.GetType();
    if (thatType!=thisType) {
        return false;
    }
    Object thisObj = (Object)this;
    Object thisResult, thatResult;
    // if there are no GC references in this object we can avoid reflection 
    // and do a fast memcmp
    if (CanCompareBits(this))
        return FastEqualsCheck(thisObj, obj);
    FieldInfo[] thisFields = thisType.GetFields(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
    for (int i=0; i<thisFields.Length; i++) {
        thisResult = ((RtFieldInfo)thisFields[i]).UnsafeGetValue(thisObj);
        thatResult = ((RtFieldInfo)thisFields[i]).UnsafeGetValue(obj);
    
        if (thisResult == null) {
            if (thatResult != null)
                return false;
        }
        else
        if (!thisResult.Equals(thatResult)) {
            return false;
        }
    }
    return true;
}
```

If your struct contains reference types, it will be compared via reflection! If there are no references, the comparison will be much faster, as it will use two helper methods defined in unmanaged code:

```c#
[System.Security.SecuritySafeCritical]  // auto-generated
[ResourceExposure(ResourceScope.None)]
[MethodImplAttribute(MethodImplOptions.InternalCall)]
private static extern bool CanCompareBits(Object obj);

[System.Security.SecuritySafeCritical]  // auto-generated
[ResourceExposure(ResourceScope.None)]
[MethodImplAttribute(MethodImplOptions.InternalCall)]
private static extern bool FastEqualsCheck(Object a, Object b);
```

The code is in `src/vm/comutilnative.cpp`:

```c++
// Return true if the valuetype does not contain pointer and is tightly packed
FCIMPL1(FC_BOOL_RET, ValueTypeHelper::CanCompareBits, Object* obj)
{
    FCALL_CONTRACT;

    _ASSERTE(obj != NULL);
    MethodTable* mt = obj->GetMethodTable();
    FC_RETURN_BOOL(!mt->ContainsPointers() && !mt->IsNotTightlyPacked());
}
FCIMPLEND

FCIMPL2(FC_BOOL_RET, ValueTypeHelper::FastEqualsCheck, Object* obj1, Object* obj2)
{
    FCALL_CONTRACT;

    _ASSERTE(obj1 != NULL);
    _ASSERTE(obj2 != NULL);
    _ASSERTE(!obj1->GetMethodTable()->ContainsPointers());
    _ASSERTE(obj1->GetSize() == obj2->GetSize());

    TypeHandle pTh = obj1->GetTypeHandle();

    FC_RETURN_BOOL(memcmp(obj1->GetData(),obj2->GetData(),pTh.GetSize()) == 0);
}
FCIMPLEND
```

Both methods on `MethodTable` check specific flags so are reasonably fast.

# Part 3. Immutable reference types in general and `string` in particular

A reference type that is immutable should behave like it possesses no object identity. Obviously, this will not protect you from `ReferenceEquals`, but all other comparison mechanisms should be modified. Let's see how `string` does that.

## A useful aside

`string` is a special type and is interned by the runtime. That's why `ReferenceEquals` might sometimes return true when you compare different string objects.

## `virtual Equals`

```c#
// Determines whether two strings match.
[ReliabilityContract(Consistency.WillNotCorruptState, Cer.MayFail)]
public override bool Equals(Object obj)
{
    if (this == null)                        //this is necessary to guard against reverse-pinvokes and
        throw new NullReferenceException();  //other callers who do not use the callvirt instruction

    String str = obj as String;
    if (str == null)
        return false;

    if (Object.ReferenceEquals(this, obj))
        return true;

    if (this.Length != str.Length)
        return false;

    return EqualsHelper(this, str);
}
```

The implementation is straightforward: there are some null check, a reference equality shortcut, a domain specific shortcut and the "meat" of the method is implemented in a helper.

`EqualsHelper` compares the contents of two strings character by character using some clever but straightforward optimizations.

## `static Equals` and operator `==`

Framework design guidelines say that all your operators should have equivalent static methods to be usable from languages that do not support operators. This is what `string` does:

```c#
public static bool operator == (String a, String b) {
   return String.Equals(a, b);
}

[Pure]
public static bool Equals(String a, String b) {
    if ((Object)a==(Object)b) {
        return true;
    }
    if ((Object)a==null || (Object)b==null) {
        return false;
    }
    if (a.Length != b.Length)
        return false;
    return EqualsHelper(a, b);
}
```

Again, there are some obvious null checks, a reference equality shortcut, a domain specific shortcut and a helper call.

# `IEquatable<T> and `EqualityComparer<T>.Default`

Why did Microsoft implement a separate interface with yet another `Equals(T)` method if all objects support `Equals(object)` anyway? The answer is performance, of course. Every call to `Equals(object)` includes a runtime type check, it's especially problematic for value types which have to be boxed. If your generic type does lots of comparisons of its parameterized members, you need a way to avoid these checks. Let's see how `List<T>` does that.

```c#
public bool Contains(T item) {
    if ((Object) item == null) {
        for(int i=0; i<_size; i++)
            if ((Object) _items[i] == null)
                return true;
        return false;
    }
    else {
        EqualityComparer<T> c = EqualityComparer<T>.Default;
        for(int i=0; i<_size; i++) {
            if (c.Equals(_items[i], item)) return true;
        }
        return false;
    }
}
```

The first half of the method is the fast past for null checks, where reference equality is the way to go. But what's going on in the second part? What's this `EqualityComparer<T>.Default`? It's a cached result of a call to `CreateComparer`, the source of which is shown below:

```c#
[System.Security.SecuritySafeCritical]  // auto-generated
private static EqualityComparer<T> CreateComparer() {
    Contract.Ensures(Contract.Result<EqualityComparer<T>>() != null);
    RuntimeType t = (RuntimeType)typeof(T);
    // Specialize type byte for performance reasons
    if (t == typeof(byte)) {
        return (EqualityComparer<T>)(object)(new ByteEqualityComparer());
    }
    // If T implements IEquatable<T> return a GenericEqualityComparer<T>
    if (typeof(IEquatable<T>).IsAssignableFrom(t)) {
        return (EqualityComparer<T>)RuntimeTypeHandle.CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(GenericEqualityComparer<int>), t);
    }
    // If T is a Nullable<U> where U implements IEquatable<U> return a NullableEqualityComparer<U>
    if (t.IsGenericType && t.GetGenericTypeDefinition() == typeof(Nullable<>)) {
        RuntimeType u = (RuntimeType)t.GetGenericArguments()[0];
        if (typeof(IEquatable<>).MakeGenericType(u).IsAssignableFrom(u)) {
            return (EqualityComparer<T>)RuntimeTypeHandle.CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(NullableEqualityComparer<int>), u);
        }
    }

    // See the METHOD__JIT_HELPERS__UNSAFE_ENUM_CAST and METHOD__JIT_HELPERS__UNSAFE_ENUM_CAST_LONG cases in getILIntrinsicImplementation
    if (t.IsEnum) {
        TypeCode underlyingTypeCode = Type.GetTypeCode(Enum.GetUnderlyingType(t));
        // Depending on the enum type, we need to special case the comparers so that we avoid boxing
        // Note: We have different comparers for Short and SByte because for those types we need to make sure we call GetHashCode on the actual underlying type as the 
        // implementation of GetHashCode is more complex than for the other types.
        switch (underlyingTypeCode) {
            case TypeCode.Int16: // short
                return (EqualityComparer<T>)RuntimeTypeHandle.CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(ShortEnumEqualityComparer<short>), t);
            case TypeCode.SByte:
                return (EqualityComparer<T>)RuntimeTypeHandle.CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(SByteEnumEqualityComparer<sbyte>), t);
            case TypeCode.Int32:
            case TypeCode.UInt32:
            case TypeCode.Byte:
            case TypeCode.UInt16: //ushort
                return (EqualityComparer<T>)RuntimeTypeHandle.CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(EnumEqualityComparer<int>), t);
            case TypeCode.Int64:
            case TypeCode.UInt64:
                return (EqualityComparer<T>)RuntimeTypeHandle.CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(LongEnumEqualityComparer<long>), t);
        }
    }
    // Otherwise return an ObjectEqualityComparer<T>
    return new ObjectEqualityComparer<T>();
}
```

Okay, this is a long and complex method, but it's called only once per type during the execution of the program, so it's a huge gain. What's going on in it, anyway?

It uses reflection to determine the runtime type of its generic argument and returns a different `EqualityComparer` for particular types:

- `ByteEqualityComparer` for `byte`
- `GenericEqualityComparer<T>` for types that implement `IEquatable<T>`
- `NullableEqualityComparer<U>` for types that are `Nullable<IEquatable<U>>`
- various `EnumEqualityComparer`s for enum types, depending on their underlying type
- `ObjectEqualityComparer<T>` for all other cases

What do these comparers do? Again, nothing truly surprising.

`GenericEqualityComparer<T>` uses `IEquatable<T>.Equals`:

```c#
[Pure]
public override bool Equals(T x, T y) {
    if (x != null) {
        if (y != null) return x.Equals(y);
        return false;
    }
    if (y != null) return false;
    return true;
}
```

`NullableEqualityComparer<T>` uses `IEquatable<T>.Equals` too:

```c#
[Pure]
public override bool Equals(Nullable<T> x, Nullable<T> y) {
    if (x.HasValue) {
        if (y.HasValue) return x.value.Equals(y.value);
        return false;
    }
    if (y.HasValue) return false;
    return true;
}
```

Finally, ObjectEqualityComparer uses `Equals(object)`:

```c#
[Pure]
public override bool Equals(T x, T y) {
    if (x != null) {
        if (y != null) return x.Equals(y);
        return false;
    }
    if (y != null) return false;
    return true;
}
```

This ensures that `List<T>` or any other type that uses `EqualityComparer<T>.Default` will use `IEquatable.Equals(T)` if `T` implements `IEquatable<T>` and `Equals(object)` if it doesn't.

# `IStructuralEquatable` and `StructuralComparisons.StructuralEqualityComparer`

What? **Another** way to compare two types? Why on Earth would you want another one? `IStructuralEquatable` is necessary for reference types that possess object identity, but are often equated based on their contents. Arrays are one such class. You should implement `IStructuralEquatable` explicitly, since it's an uncommon operation. Also not that apart from the other object, `IStructuralEquatable.Equals` takes a `IEqualityComparer` parameter. `IEqualityComparer` is like `IEqualityComparer<T>`, but worse. It cannot be a generic type because some structurally comparable types, like `System.Tuple`, are heterogenous. Tecnically, you could implement interfaces `IEqualityComparer<T1, T2, T3>` and so on that took even more parameters in the `Equals` method, but this would be a very narrow use case.

This is how `System.Array` implements the method:

```c#
Boolean IStructuralEquatable.Equals(Object other, IEqualityComparer comparer) {
    if (other == null) {
        return false;
    }
    if (Object.ReferenceEquals(this, other)) {
        return true;
    }
    Array o = other as Array;
    if (o == null || o.Length != this.Length) {
        return false;
    }
    int i = 0;
    while (i < o.Length) {
        object left = GetValue(i);
        object right = o.GetValue(i);
        if (!comparer.Equals(left, right)) {
            return false;
        }
        i++;
    }
    return true;
}
```

Again, the method is rather straightforward. Some null checks, two shortcuts and an elementwise check.

## `StructuralComparisons.StructuralEqualityComparer`

But what if you have to compare two arrays of arrays of arrays, i.e., values of type `SampleClass[][][]`? What kind of comparer would you pass to `IStructuralEquatable.Equals`? Enter `StructuralComparisons.StructuralEqualityComparer`.

```c#
public new bool Equals(Object x, Object y) {
    if (x != null) {

        IStructuralEquatable seObj = x as IStructuralEquatable;

        if (seObj != null){
            return seObj.Equals(y, this);
        }

        if (y != null) {
            return x.Equals(y);
        } else {
            return false;
        }
    }
    if (y != null) return false;
    return true;
}
```

It simply calls `IStructuralEquatable.Equals(object)` if the lhs is `IStructuralEquatable` and `object.Equals(object)` if it isn't. Note that it cannot call `IEquatable<T>.Equals`, so it loses a tiny bit of performance, but since it already uses a runtime cast, it's an acceptable loss.

# The properties of equality and `GetHashCode`

Implementations of `Equals` method must satisfy the mathematical properties of equality when both operands are of the same runtime type:

- reflexivity: for any `a`, `Equals(a, a)` must return true
- symmetry: for any `a`, `b`, `Equals(a, b)` and `Equals(b, a)` must return the same result
- transitivity: for any `a`, `b`, `c`, `Equals(a, b) && Equals(b, c)` and `Equals(a, c)` must return the same result

`GetHashCode` is a performance shortcut method for equality testing. This means that:

- it must be fast: if it's slower than `Equals`, there's not much use in it
- it must be deterministic: `Equals(a.GetHashCode(), a.GetHashCode())` must always return true
- it must be reliable: it mustn't throw exceptions
- it must be reflect the equality: if `Equals(a, b)` returns `true`, `Equals(a.GetHashCode(), b.GetHashCode())` must also return true
- its result *should* be uniformly distributed

Every time you override the `Equals` method you must also override `GetHashCode` to satisfy the above requirements. Obviously, an implementation like the one below is a valid, but completely useless one:

```c#
public override int GetHashCode() => 0;
```

A good implementation of `GetHashCode` is provided by Jon Skeet (who else) here: http://stackoverflow.com/a/263416/3106795

# When `Equals` and `==` are not equal

In general all equality methods must produce consistent results. However, numeric value types are a very well-known exception. Numeric types are promoted to the widest type by `==` while `Equals` doesn't do that:

```c#
Console.WriteLine(1.0 == 1.0f);      //true
Console.WriteLine(1.0f == 1.0);      //true
Console.WriteLine(1.0f.Equals(1.0)); //false
Console.WriteLine(1.0.Equals(1.0f)); //true
```

Similarly, IEEE 754:2008 defines non-reflective operations on floating point types, so operators follow the standard while `Equals` matches the general rules:

```c#
Console.WriteLine(double.NaN == double.NaN);      //false
Console.WriteLine(double.NaN.Equals(double.NaN)); //true
```

When implementing custom types you should avoid such discrepancies in behaviour if they aren't absolutely necessary.

# General rules on implementing custom equality

## Reference types with object identity

Don't do anything, you are covered by the runtime. You can theoretically add the following implementation of `IEquatable<T>.Equals`, but it's 100% useless:

```c#
public Equals(YourClass other) => this==other;
```

### Reference container types with object identity

If your type is a container of sorts, implement `IStructuralEquatable.Equals` and `IStructuralEquatable.GetHashCode` explicitly.

## Value types or immutable reference types without object identity

0. Indicate that your type implements `IEquatable<YourValueType>`.

1. Implement `Equals(YourValueType)` that compares its two parameters using their members' `Equals` methods. If there are generic members, use `EqualityComparer<T>.Default.Equals` instead. Make sure your method is reflexive, symmetric and transitive.

2. Implement `Equals(object)` that checks that its argument is an instance of `YourValueType` and delegates the rest of the job to the previous method.

3. Override `GetHashCode` to match the behavior of `Equals(object)`. Make sure your method is fast, deterministic and reliable.

4. Implement a static `Equals(YourValueType a, YourValueType b)` method that compares its two parameters using their members' `Equals` methods or other applicable methods if you're comparing `double`s or other unusual types.

5. Implement operator `==` that calls `Equals(YourValueType a, YourValueType b)`. Implement operator '!=` that calls `!Equals(YourValueType a, YourValueType b)`.

If your type doesn't contain any unusual types, you can and should use the same static helper method `Equals(YourValueType a, YourValueType b)` for all other equality methods.
