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
			// do something with it
		}
	}
}
```

### Optional parameter annotation

`@Optional`: the parameter can be omitted in which case the value received will be the neutral type value (i.e. `null`, `0` or `false`).

```java
public class Test {	

	public static void sayHello(@Optional String who) {
		if (who == null) who = "world";
		System.out.println("Hello " + who + "!");
	}
	
	public static void main(String[] args) {
		sayHello(); // -> Hello world!
	}
}
```

### Conditional method annotations

* `@Conditional`, `@ConditionalCallerField`: on a method, allows to condition calls to a specific method depending on a global constant and/or a constant on the caller type. Note: parameters will __not__ be evaluated.

Useful for logging or assertion methods. Corresponds to the `[Conditional]` attribute in C#.

```java
public class Test {
	
	@Conditional(false)
	public static void disabled(Object arg) {
	}
	
	@Conditional(true)
	public static void enabled(Object arg) {
	}
	
	public static final boolean EnableMe = true;
	
	@Conditional(false)
	@ConditionalCallerField("EnableMe")
	public static void enabledFromTest(Object arg) {
	}
	
	public static void main(String[] args) {
		var hello = "Hello";
		disabled(hello = hello + " world!");
		System.out.println(hello); // -> Hello
		enabled(hello = hello + " world!");
		System.out.println(hello); // -> Hello world!
		enabledFromTest(hello = hello + " (again)");
		System.out.println(hello); // -> Hello world! (again)
	}
}
```

### Synthetic parameters annotations

Extremely useful for logging stuff:

* `@CallerFile`, `@CallerLine`, `@CallerClass`, `@CallerMethod`: on a method parameter, replaced by the corresponding value in the caller's context.

(similar to the C# attributes `[CallerFilePath]`, `[CallerMemberName]`, ...).

* `@ArgName`, `@ArgNames`: on a method parameter, replaced with the caller-side representation for the next (non-synthetic) argument(s).

Note: if the caller expression is a method/constructor call, the compiler will try to find a symbolic name in the arguments of the invocation (recursively).

(similar to the `nameof` operator in C#).

```java
public class Test {

	public static void log(@CallerFile String file, @CallerLine int line, @CallerMethod String method, String message, @ArgNames String[] names, Object ...args) {
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

Note: these are "synthetic" parameters, in the sense that these parameters are invisible to the caller (the completion system has been modified to hide these parameters).

The big advantage of this approach (instead of using optional parameters) is that these parameters can appear before regular parameters which means this can be combined with varargs (very useful for logging). This is a big advantage over the C# equivalents with use optional parameters.

Limitation: it does not interact well with generic methods (not really an issue in practice).

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
3. Add `plugins/org.eclipse.jdt.annotation_xxx.jar` from the Eclipse installation to the project (the one with the highest version and date)

## Implementation notes

The feature patch is produced from a fork of Eclipse JDT [core](https://github.com/philippejer/eclipse.jdt.core/tree/extensions) and [UI](https://github.com/philippejer/eclipse.jdt.ui/tree/extensions) repositories.

Unlike project Lombok, this is not intended to be portable to other compilers or IDEs. This allows to insert the required hooks into the JDT code directly instead of using complicated code injection via reflection. Also, this is not intended as a "clean" implementation, i.e. it is not tightly coupled with the JDT code. This means it can be merged easily with the official repositories.

Note: since this is non-standard Java extensions, code using these features can only be compiled by a patched Eclipse JDT. However the resulting bytecode is perfectly normal and will be optimized at runtime by the JIT compiler.