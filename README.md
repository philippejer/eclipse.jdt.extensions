# JDT Extensions

This is a patch for Eclipse JDT to add support a few Java extensions, inspired by
[Project Lombok](https://projectlombok.org).

Important: source files using any of these features can only be compiled by a patched Eclipse IDE.

## List of extensions

### `var` keyword

Allows to use C#-style type inference using the `var` keyword (for local variables and collection iterators).

Implementation extracted from Project Lombok but integrated directly into the JDT.

```java
public class Test {	

	public static void test() {
		var list = new LinkedList<String()>();
		
		list.add("hello");
		list.add(" world!");
		
		for (var s: list) {
			System.out.print(s);
		}
		System.out.println();
	}
}
```

### Default argument values

Same as C++ or C# default argument values, allows to specify default values for method parameters.

Note: this will automatically generate appropriate method overload(s), just like this would be done in "plain" Java.
Any valid expression valid in the context of the overloaded method can be used, including referencing other arguments
(provided they do not have a default value).

```java
public class Test {	

	public static void sayHello(String who = "world") {
		System.out.println("Hello " + who + "!");
	}
	
	// The following method will be automatically generated
	public static void sayHello() {
		sayHello("world");
	}
	
	public static void main(String[] args) {
		sayHello(); // -> Hello world!
	}
}
```

### Caller annotations

Inspired by C#, mostly intended for diagnostics:

* `@CallIf`: on a method, allows to disable all calls to a specific method (including parameter evaluation!).

Useful for logging or assertion methods. Corresponds to the `Conditional` C# attribute.

```java
public class Test {
	
	@CallIf(false)
	public static void disabled(Object arg) {
	}
	
	@CallIf(true)
	public static void enabled(Object arg) {
	}
	
	public static void main(String[] args) {
		var hello = "Hello";
		disabled(hello = hello + " world!");
		System.out.println(hello); // -> Hello
		enabled(hello = hello + " world!");
		System.out.println(hello); // -> Hello world!
	}
}
```

* `@CallFile`, `@CallLine`, `@CallClass`, `@CallMethod`: on a method parameter, replaced by the corresponding value in the caller's context.

(similar to the `Caller...` attributes in C#).

* `@CallName`, `@CallNames`: on a method parameter, replaced with the caller-side representation for the next (non-synthetic) argument(s).

Note: if the caller expression is a method/constructor call, the compiler will try to find a symbolic name in the arguments of the invocation (recursively).

(similar to the `nameof` operator in C#).

```java
public class Test {

	public static void log(@CallFile String file, @CallLine int line, @CallMethod String method, String message, @CallNames String[] names, Object ...args) {
		var output = new StringBuilder();
		output.append("(").append(new File(file).getName()).append(":").append(line).append("): ").append(method).append(": ").append(message).append(": ");
		for (int i = 0; i < args.length; i++) {
			if (i > 0) output.append(' ');
			output.append(names[i]).append('=').append(args[i]);
		}
		System.out.println(output);
	}
	
	public static void main(String[] args) {
		var v1 = "hello";
		var v2 = 42;
		log("here you are", v1, String.format("0x%X", v2)); // -> (Test.java:129): main: here you are: v1=hello v2=0x2A
	}
}
```

Note: these are "synthetic" annotations, in the sense that the arguments are invisible to the caller (the completion system has been modified to hide these parameters).

The big advantage of this approach (instead of using optional arguments) is that these parameters can appear before regular parameters and so this can be combined with variable number of arguments (very useful for logging). This is a big advantage over C# with uses optional arguments.

Limitation: it does not work on generic method (it could be fixed but that's not an issue in practice since this is mostly useful for logging stuff).

### Public by default

This sets the default access protection to public instead of Package-Private for a project. To enable, simply add to following to `${Workspace}/${Project}/.settings/org.eclipse.jdt.core.prefs`:

```
org.eclipse.jdt.core.compiler.extensions.publicByDefault=enabled
```

Not for everyone obviously but often on personal project you just want to write code and worry about encapsulation later.

```java
class Hello {

	static void main(String[] args) { // would not run normally since not public
		System.out.println("Hello!");
	}
}
```

## Installation

To install the patch:

1. Download the zip archive `eclipse.jdt.extensions.zip`.
2. Use "Help" --> "Install New Software" --> "Add" --> "Archive".

## Implementation notes

The feature patch is produced from a fork of Eclipse JDT [core](https://github.com/philippejer/eclipse.jdt.core/tree/extensions) and [UI](https://github.com/philippejer/eclipse.jdt.ui/tree/extensions) repositories.

Unlike project Lombok, this is not intended to be portable to other compilers or IDEs. This allows to insert the required hooks into the JDT code directly instead of using complicated code injection via reflection. Also, this is not intended as a "clean" implementation, i.e. it is not tightly coupled with the JDT code. This means it can be merged easily with the official repositories.

Note: since this is non-standard Java extensions, code using these features can only be compiled by a patched Eclipse JDT. However the resulting bytecode is perfectly normal and will be optimized at runtime by the JIT compiler.