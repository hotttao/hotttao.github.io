# go reflect


go 反射机制

<!-- more -->

## 1. 反射机制
反射是一个复杂的内省技术。所谓内省即可以动态获取变量的类型，值，以及方法属性等元数据。需要反射的根本原因是，很多时候我们在编程时，并不能确定输入的具体类型，需要我们动态去判断。

Go语言提供的反射机制，能够让我们在运行时更新变量和检查它们的值、调用它们的方法和它们支持的内在操作，而不需要在编译时就知道这些变量的具体类型。也可以让我们将类型本身作为第一类的值类型处理。

GO 中有两个至关重要的API是使用反射机制实现：
1. fmt包提供的字符串格式功能
2. 类似encoding/json和encoding/xml提供的针对特定协议的编解码功能。

本节我们就来看看如何使用 Go 的反射机制，以及上述两个包使用 reflect 的方式。


## 2. Reflect API
反射是由 reflect 包提供的。 它定义了两个重要的类型, Type 和 Value

### 2.1 Type
Type 是一个接口类型，唯一能反映 reflect.Type 实现的是接口的类型描述信息。

我们在接口一节说过，接口的值，由两个部分组成，一个具体的类型和那个类型的值。它们被称为接口的动态类型和动态值。对于像Go语言这种静态类型的语言，类型是编译期的概念；因此一个类型不是一个值。在我们的概念模型中，一些提供每个类型信息的值被称为类型描述符，比如类型的名称和方法。在一个接口值中，类型部分代表与之相关类型的描述符。而 reflect.Type 的实现方式就与接口中的类型描述符类似。

reflect.Type 有许多办法来区分类型以及检查它们的组成部分, 例如一个结构体的成员或一个函数的参数等。

#### TypeOf
reflect.TypeOf 接受任意的 interface{} 类型, 并以reflect.Type形式返回一个动态类型的接口值。reflect.Type 满足 fmt.Stringer 接口。 fmt.Printf 提供的 %T 参数, 内部就是使用 reflect.TypeOf 来输出接口的动态类型。

```Go
t := reflect.TypeOf(3) // a reflect.Type
fmt.Println(t.String()) // "int"
fmt.Println(t) // "int"

fmt.Printf("%T\n", 3) // "int"
```


### 2.2 Value
reflect.Value 可以装载任意类型的值。函数 reflect.ValueOf 接受任意的 interface{} 类型, 并返回一个装载着其动态值的 reflect.Value。和 reflect.Type 类似, reflect.Value 也满足 fmt.Stringer 接口, 但是除非 Value 持有的是字符串, 否则 String 方法只返回其类型. 而使用 fmt 包的 %v 标志参数会对 reflect.Values 特殊处理.

```Go
v := reflect.ValueOf(3) // a reflect.Value
fmt.Println(v) // "3"
fmt.Printf("%v\n", v) // "3"
fmt.Println(v.String()) // NOTE: "<int Value>"
```

对 Value 调用 Type 方法将返回具体类型所对应的 reflect.Type:
```Go
t := v.Type() // a reflect.Type
fmt.Println(t.String()) // "int"
```

### 2.3 Value 与 interface{}
reflect.ValueOf 的逆操作是 reflect.Value.Interface 方法. 它返回一个 interface{} 类型，装载着与 reflect.Value 相同的具体值。

```Go
v := reflect.ValueOf(3) // a reflect.Value
x := v.Interface()      // an interface{}
i := x.(int)            // an int
fmt.Printf("%d\n", i)   // "3
```

reflect.Value 和 interface{} 都能装载任意的值. 所不同的是, 一个空的接口隐藏了值内部的表示方式和所有方法, 因此只有我们知道具体的动态类型才能使用类型断言来访问内部的值(就像上面那样),内部值我们没法访问. 相比之下, 一个 Value 则有很多方法来检查其内容, 无论它的具体类型是什么。

与 `switch x := x.(type)` 相比 `reflect.Value.Kind` 返回的数据类型是有限的:
1. Bool, String 和 所有数字类型的基础类型
2. Array 和 Struct 对应的聚合类型;
3. Chan, Func, Ptr, Slice, 和 Map 对应的引用类型;
4. interface 类型;
5. 还有表示空值的 Invalid 类型 (空的 reflect.Value 的 kind 即为 Invalid.)

 Kind 只关心底层表示, 所有的具名类型都会归属到对应的原始类型之上。

 ### 2.4 示例

```Go
func Display(name string, x interface{}) {
	fmt.Printf("Display %s (%T):\n", name, x)
	display(name, reflect.ValueOf(x))
}

func formatAtom(v reflect.Value) string {
	switch v.Kind() {
		case reflect.Invalid:
			return "invalid"
		case reflect.Int, reflect.Int8, reflect.Int16,
		   reflect.Int32, reflect.Int64:
			return strconv.FormatInt(v.Int(), 10)
		case reflect.Uint, reflect.Uint8, reflect.Uint16,
		   reflect.Uint32, reflect.Uint64, reflect.Uintptr:
			return strconv.FormatUint(v.Uint(), 10)
		// ...floating‐point and complex cases omitted for brevity...
		case reflect.Bool:
			return strconv.FormatBool(v.Bool())
		case reflect.String:
			return strconv.Quote(v.String())
		case reflect.Chan, reflect.Func, reflect.Ptr, reflect.Slice, reflect.Map:
			return v.Type().String() + " 0x" +
				strconv.FormatUint(uint64(v.Pointer()), 16)
		default: // reflect.Array, reflect.Struct, reflect.Interface
			return v.Type().String() + " value"
	}
}


func display(path string, v reflect.Value) {
	switch v.Kind() {
		case reflect.Invalid:
			fmt.Printf("%s = invalid\n", path)
		case reflect.Slice, reflect.Array:
			for i := 0; i < v.Len(); i++ {
				display(fmt.Sprintf("%s[%d]", path, i), v.Index(i))
}
		case reflect.Struct:
			for i := 0; i < v.NumField(); i++ {
				fieldPath := fmt.Sprintf("%s.%s", path, v.Type().Field(i).Name)
				display(fieldPath, v.Field(i))
}
		case reflect.Map:
			for _, key := range v.MapKeys() {
				display(fmt.Sprintf("%s[%s]", path,
					formatAtom(key)), v.MapIndex(key))
}
		case reflect.Ptr:
				if v.IsNil() {
					fmt.Printf("%s = nil\n", path)
				} else {
					display(fmt.Sprintf("(*%s)", path), v.Elem())
}
		case reflect.Interface:
			if v.IsNil() {
				fmt.Printf("%s = nil\n", path)
			} else {
				fmt.Printf("%s.type = %s\n", path, v.Elem().Type())
				display(path+".value", v.Elem())
		}
		default: // basic types, channels, funcs
			fmt.Printf("%s = %s\n", path, formatAtom(v))
	}
}
```

