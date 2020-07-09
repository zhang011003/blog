# java8 lambda

一直对lambda表达式不是很了解，这里先记录一下方法引用

以下为几种引用方式

1. **对象::实例方法**，将lambda的参数当做方法的参数使用

2. **类::静态方法**，将lambda的参数当做方法的参数使用

3. **类::实例方法**，将lambda的第一个参数当做方法的调用者，其他的参数作为方法的参数

4. **类::new**，将lambda的参数当做构造方法参数，最后一个参数为返回的类。如果构造方法无参数，则lambda参数只有一个，就是返回的类

示例：

```java
 // 类名::静态方法
 // lambda参数为对应的方法参数
 Function<Integer, String> f1 = String::valueOf;
 // 与下面的声明相同
 IntFunction<String> f2 = String::valueOf;

 // 类名::实例方法                                    
 // 第一个参数为对象实例，第二个参数为对应的方法参数                    

 BiPredicate<String, String> p = String::equals;

 // 类名::new                                 
 // 无参数构造方法                                 
 Supplier<List> s1 = ArrayList<String>::new;

 // 类名::new                                            
 // 有参数构造方法。第一个参数为构造方法参数，第二个参数为返回的类                    
 Function<Integer, List> f3 = ArrayList<String>::new;  

 // 类名::实例方法                                
 // 第一个参数为实例，第二个参数为对应的方法参数                  
 Function<List, Integer> f4 = List::size;    

 // 类名::实例方法                             
 // 对于无参数的实例方法，第一个参数为返回的值                  
 Predicate<String> p2 = String::isEmpty; 

 // 类名::实例方法                                                
 // 对于多个参数的实例方法，第一个参数为实例，第二个参数为对应的方法参数，第三个参数为对应的方法返回值       
 BiFunction<String,Integer,String> f5 = String::substring;  
```
