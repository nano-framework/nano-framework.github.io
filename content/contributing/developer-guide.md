# Developer Guideline

We're working on it! Stay tuned! Raise PR and that will help us finding good recommendations.

Till then here follows some unsorted tips.

## How to call native code from managed code

Assuming you want to call from nanoframework's mscorlib (source can be found in lib-CoreLibrary repository) C# code (e.g. System.Number class) some implementation you would like to place in it's nanoCLR (source in nf-interpreter repository) C++ code. Follow steps below:

1. Build the nf-CoreLibrary solution without making any changes.
2. Copy the ```nanoFramework.CoreLibrary\bin\Debug\Stubs``` folder somewhere for later use.
3. Declare your C++ function in your C# class:

```
[MethodImpl(MethodImplOptions.InternalCall)]
private static extern String FormatNative(
   Object value,
   bool isInteger,
   String format,
   String numberDecimalSeparator,
   String negativeSign,
   String numberGroupSeparator,
   int[] numberGroupSizes);
```

2. Add code which calls the function above as you wish.
3. Build the solution.
4. Compare the ```nanoFramework.CoreLibrary\bin\Debug\Stubs``` folder's actual state with the saved one. The files which should have changed:
- ```corlib_native.cpp```
- ```corlib_native.h```
- your class's C++ counterpart, ```corlib_native_System_Number.cpp``` in the my example
5. Apply the changes you found to same files under ```nf-interpreter/src/CLR/CorLib```. DO NOT overwrite the files there! The files under nf-interpreter may have additional declarations, etc. Copy over the diff meaningfully!
6. You will find that a stub for the function you declared above will be generated with this signature:

```
HRESULT Library_corlib_native_System_Number::
    FormatNative___STATIC__STRING__OBJECT__BOOLEAN__STRING__STRING__STRING__STRING__SZARRAY_I4(CLR_RT_StackFrame &stack)
```

7. Now you can implement your function.

## How to handle in C++ parameter values received from C# call

Lets see the example method discussed in (#How-to-call-native-code-from-managed-code) tip. The generated (by lib-CoreLibrary solution build, where you declared your needs as an ```extern``` function) C++ stub will have a ```CLR_RT_StackFrame &stack``` parameter.
The values can be accessed as follows:

### ```object value```

```
// get ref to value container
CLR_RT_HeapBlock *value;
value = &(stack.Arg0());

// perform unboxing operation if necessary
CLR_RT_TypeDescriptor desc;
NANOCLR_CHECK_HRESULT(desc.InitializeFromObject(*value));
NANOCLR_CHECK_HRESULT(value->PerformUnboxing(desc.m_handlerCls));

// get the CLR_DataType of the value in container, like DATATYPE_U1 for a Byte or DATATYPE_R8 for Double
CLR_DataType dataType = value->DataType();

// retrieving the real value depends on dataType above
int32_t int32Value = value->NumericByRef().s4;
```

### ```bool isInteger```

```
// get value
bool isInteger;
isInteger = (bool)stack.Arg1().NumericByRef().u1;
```

### ```String format```

```
// get value
char *format;
format = (char *)stack.Arg2().RecoverString();
```

### ```int[] numberGroupSizes```

```
// get ref to value container
CLR_RT_HeapBlock_Array *numberGroupSizes;
numberGroupSizes = stack.Arg6().DereferenceArray();

// get number of elements
CLR_UINT32 numOfElements = numberGroupSizes->m_numOfElements;

// get the 5th element
// cast necessary, because GetElement declared as CLR_INT8*, 
// but the C# code call placed items of Int32 type into array.
int the5thEelement = *((CLR_INT32 *)numberGroupSizes->GetElement(5));
```

## Returning from C++ function for C# code

Values should be returned via the ```CLR_RT_StackFrame &stack``` parameter.

### String

```
char * ret;
// ... assign value to ret ...

// use helper methods to set return value
NANOCLR_SET_AND_LEAVE(stack.SetResult_String(ret));
```

