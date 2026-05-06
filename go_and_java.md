🚀 Java ↔ Go Syntax Mapping Table
Concept	Java	Go
Program Entry	public static void main(String[] args)	func main()
Print	System.out.println("Hi")	fmt.Println("Hi")
Variable Declaration	int a = 10;	var a int = 10
Short Declaration	❌	a := 10
Type Inference	Limited (var)	Strong (:=)
Data Types	int, long, double, String	int, int64, float64, string
If Condition	if (a > 5)	if a > 5
For Loop	for(int i=0;i<n;i++)	for i := 0; i < n; i++
While Loop	while(condition)	for condition
Array	int[] arr = {1,2}	arr := []int{1,2}
Dynamic Array	ArrayList<>	slice (append)
Add Element	list.add(10)	list = append(list, 10)
Map / HashMap	Map<K,V>	map[K]V
Insert in Map	map.put("a",1)	map["a"] = 1
Check Key Exists	map.containsKey("a")	val, ok := map["a"]
Function	int add(int a,int b)	func add(a int,b int) int
Return Multiple Values	❌ (use array/object)	return a,b
Class	class Person {}	type Person struct {}
Method	Inside class	func (p Person) method()
Constructor	Person()	No direct (use function)
Interface	interface A {}	type A interface {}
Implementation	implements keyword	Implicit
Inheritance	extends	❌ (composition used)
Error Handling	try-catch	if err != nil
Custom Error	Exception class	errors.New()
Threading	Thread / Executor	goroutine (go func())
Synchronization	synchronized	sync.Mutex
String Length	s.length()	len(s)
Pointers	❌	&a, *p
Null Handling	null	nil
Defer (finally)	finally	defer
Package Import	import java.util.*	import "fmt"
🔥 High-Impact Interview Differences
Feature	Java	Go
OOP Style	Heavy OOP	Lightweight (struct + funcs)
Concurrency	Complex (threads)	Simple (goroutines)
Error Handling	Exceptions	Explicit error return
Inheritance	Yes	No (composition)
Performance	JVM-based	Near C-level
Learning Curve	Medium	Easy to start, tricky in depth
⚡ Quick Example Mapping
Task	Java	Go
Loop array	for(int x: arr)	for _, x := range arr
Map iteration	for(Map.Entry e : map.entrySet())	for k,v := range map
Function call	add(1,2)	add(1,2)
Ignore value	❌	_ (blank identifier)
