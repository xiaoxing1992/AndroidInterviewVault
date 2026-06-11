---
tags: [java, generics, type-erasure, kotlin, android]
difficulty: A
frequency: medium
---

# Java 泛型与类型擦除

## 问题

"Java 泛型的实现原理是什么？类型擦除会带来哪些问题？`? extends T` 和 `? super T` 的区别？Kotlin 的 reified 是怎么绕过类型擦除的？"

## 核心答案（30 秒版本）

Java 泛型是**编译期语法糖**，编译后通过类型擦除将泛型参数替换为上界（通常是 Object）。擦除导致运行时无法获取泛型实际类型、不能 new T()、不能 instanceof。`? extends T` 表示上界通配符（只读），`? super T` 表示下界通配符（只写），遵循 **PECS 原则**（Producer-Extends, Consumer-Super）。Kotlin 的 `reified` 通过 **inline 函数内联 + 调用处替换具体类型** 绕过擦除，本质是编译器在每个调用点生成具体类型的字节码。

## 深入解析

### 原理层（源码级）

**1. 类型擦除过程**

```java
// 编译前
public class Box<T> {
    private T value;
    public T getValue() { return value; }
    public void setValue(T value) { this.value = value; }
}

// 编译后（擦除为 Object）
public class Box {
    private Object value;
    public Object getValue() { return value; }
    public void setValue(Object value) { this.value = value; }
}

// 有上界的擦除
public class NumberBox<T extends Number> {
    private T value;  // 擦除为 Number，不是 Object
}
```

**2. 桥方法（Bridge Method）**

```java
public class StringBox extends Box<String> {
    @Override
    public String getValue() { return "hello"; }
}

// 编译器生成桥方法保证多态正确
// 反编译后：
public class StringBox extends Box {
    public String getValue() { return "hello"; }

    // 桥方法：覆盖父类的 Object getValue()
    public /*synthetic bridge*/ Object getValue() {
        return this.getValue();  // 调用上面的 String 版本
    }
}
```

**3. 类型擦除的限制**

```java
// ❌ 不能 new T()
public <T> T create() {
    return new T();  // 编译错误，擦除后不知道具体类型
}

// ❌ 不能 instanceof 泛型
if (obj instanceof List<String>) {}  // 编译错误

// ❌ 不能创建泛型数组
T[] array = new T[10];  // 编译错误

// ✅ 通过 Class<T> 传递类型信息
public <T> T create(Class<T> clazz) throws Exception {
    return clazz.newInstance();
}
```

**4. 通配符与 PECS**

```java
// ? extends T — 上界通配符（协变）
// 只能读（Producer），不能写（编译器不知道具体子类型）
List<? extends Number> numbers = new ArrayList<Integer>();
Number n = numbers.get(0);    // ✅ 读取安全
numbers.add(1);               // ❌ 编译错误

// ? super T — 下界通配符（逆变）
// 只能写（Consumer），读取只能得到 Object
List<? super Integer> list = new ArrayList<Number>();
list.add(42);                 // ✅ 写入安全
Object obj = list.get(0);    // ✅ 但只能当 Object 用

// PECS 实践：Collections.copy 源码
public static <T> void copy(List<? super T> dest,    // Consumer: super
                             List<? extends T> src) { // Producer: extends
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));
    }
}
```

**5. 获取泛型类型的方式**

```java
// 通过匿名子类 + 反射获取（Gson/Retrofit 的做法）
Type type = new TypeToken<List<String>>(){}.getType();

// TypeToken 原理：匿名内部类保留了父类的泛型签名
// 字节码中 Signature 属性记录了泛型信息
public abstract class TypeToken<T> {
    private final Type type;

    protected TypeToken() {
        // 获取子类（匿名类）的泛型父类参数
        Type superclass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superclass).getActualTypeArguments()[0];
    }
}
```

**6. Kotlin reified 原理**

```kotlin
// reified 必须配合 inline 使用
inline fun <reified T> isInstance(obj: Any): Boolean {
    return obj is T  // 编译后替换为具体类型
}

// 调用处
val result = isInstance<String>("hello")

// 编译后等价于（内联展开）
val result = "hello" is String  // 直接使用具体类型

// 实际应用：Gson 反序列化
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)  // reified 可以获取 Class
}

// 对比 Java 必须传 Class 参数
public <T> T fromJson(String json, Class<T> classOfT);
```

### 实战层（项目经验）

**1. Retrofit 中的泛型处理**

```java
// Retrofit 通过 TypeToken 模式获取返回类型
interface ApiService {
    @GET("users")
    Call<List<User>> getUsers();  // 泛型信息保留在方法签名中
}

// Retrofit 内部通过反射获取方法返回类型
Type returnType = method.getGenericReturnType();
// returnType = Call<List<User>>
// 进一步解析得到 List<User>，传给 Converter
```

**2. Gson 序列化泛型集合**

```java
// ❌ 错误：类型擦除导致反序列化失败
List<User> users = gson.fromJson(json, List.class);
// 实际得到 List<LinkedTreeMap>

// ✅ 正确：使用 TypeToken
Type type = new TypeToken<List<User>>(){}.getType();
List<User> users = gson.fromJson(json, type);
```

**3. Android 中的泛型最佳实践**

```java
// RecyclerView.Adapter 泛型约束
public abstract class BaseAdapter<T, VH extends RecyclerView.ViewHolder> {
    private List<T> mData = new ArrayList<>();

    public void setData(List<? extends T> data) {  // PECS: extends 读取
        mData.clear();
        mData.addAll(data);
        notifyDataSetChanged();
    }
}

// ViewModel 工厂泛型
public class ViewModelFactory<VM extends ViewModel> implements ViewModelProvider.Factory {
    private final Class<VM> vmClass;

    public ViewModelFactory(Class<VM> vmClass) {
        this.vmClass = vmClass;  // 传递 Class 绕过擦除
    }

    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        return (T) vmClass.getDeclaredConstructor().newInstance();
    }
}
```

### 延伸问题

- [[反射机制与动态代理]] — 反射获取泛型信息的原理
- [[HashMap原理与ConcurrentHashMap]] — 泛型在集合框架中的应用
- [[Kotlin协程与Flow]] — Flow<T> 的类型安全设计
- [[Retrofit动态代理原理]] — Retrofit 如何解析泛型返回类型

## 记忆锚点

**泛型是编译期糖衣，擦除后变 Object；PECS = 读用 extends 写用 super；Kotlin reified 靠 inline 内联在调用处保留具体类型。**
