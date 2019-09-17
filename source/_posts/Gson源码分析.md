---
title: Gson源码分析
tags:
  - GSON
categories:
  - Java
abbrlink: ba92fa77
date: 2015-10-02 16:24:34
---
&emsp;最近研究了google开源的Json库Gson，在这里进行总结一下，应该会分为3篇博客。第一篇主要讲一下Gson的整个框架吧；第二篇主要总结一下Gson关于反射的部分;最后一篇会总结一下JsonWriter和JsonReader，主要是Json对象的处理啦。
### Gson
&emsp;Gson是可以转换Java对象为JSON表示的java库，也可以将JSON转换为Java对象，并且可以转换你没有源代码的预设的复杂对象
&emsp;现在有一些JSON转换库，但是大多数都需要你在class中设置annotation；如果没有class的源代码你就无法实现转换，而且大多数无法支持全部的Java范型。Gson把实现上述作为设计的主要目标。
&emsp;Gson的目标

 - 提供简单的toJson(),和fromJson来实现Java对象和JSON数据的相互转换
 - 运行预先存在的无法修改的对象与Json的转换
 - 支持Java范型
 - 运行用户自定义对象的结构
 - 支持复杂对象的处理
 
### 框架描述
![框架类图](http://7xjsjy.com1.z0.glb.clouddn.com/gson_GsonUML.jpg)
 &emsp;这是Gson库的类图，没有将所有类都表现上去，只是画了几个比较主要的类和我研究过的类。这篇文章就主要梳理一下这个框架，研究一下几个比较主要的函数的流程和各个对象的协作。


#### 1.GsonBuilder
 &emsp;GsonBuilder是Gson对象的Builder类啦，我们可以先看一下Gson对象的构造函数的复杂度啦,所以如果希望配置Gson,就只有使用Builder模式啦，这也是Java设计模式中所推荐的
```
Gson(final Excluder excluder, final FieldNamingStrategy fieldNamingPolicy,
      final Map<Type, InstanceCreator<?>> instanceCreators, boolean serializeNulls,
      boolean complexMapKeySerialization, boolean generateNonExecutableGson, boolean htmlSafe,
      boolean prettyPrinting, boolean serializeSpecialFloatingPointValues,
      LongSerializationPolicy longSerializationPolicy,
      List<TypeAdapterFactory> typeAdapterFactories)
```
&emsp; GsonBuilder文件中的注释也有说明：
>使用这个对象去配置你的Gson对象，当你希望修改默认配置时

&emsp;我们现在可以依次介绍一下GsonBuilder的成员变量或者说是其所依赖的类型吧。

- `Exculder  ` 是用来配置一些你不希望被转换成JSON格式的对象的成员变量的，比如你只希望自己的对象中所有public的成员变量被转换为JSON格式，那么就需要使用到这个对象，添加规则去除去所有非pulbic的成员变量。

- `LongSerializationPolicy`  

- `FieldNamingStragety`

- `InstanceCreator` 

- `TypeAdapterFactory`

#### 2.Gson
&emsp;Gson对象就是我们最常使用的对象啦，它有一系列的fromJson,toJson的成员函数供我们调用，这篇文章的一个重点就是梳理同这两类函数的逻辑。
&emsp;我们先来看一下Gson的构造函数吧。构造函数名和参数列表在前边已经列出来啦，构造函数中就是将参数列表中的对象配置为成员变量，不过要注意的是对TypeAdapterFactory的操作啦。
```

	....
    List<TypeAdapterFactory> factories = new ArrayList<TypeAdapterFactory>();

    // built-in type adapters that cannot be overridden
    factories.add(TypeAdapters.JSON_ELEMENT_FACTORY);
    factories.add(ObjectTypeAdapter.FACTORY);
    ..... //还有很多基本的TypeAdapterFactory
    // the excluder must precede all adapters that handle user-defined types
    factories.add(excluder);

    // user's type adapters
    factories.addAll(typeAdapterFactories);

    // type adapters for basic platform types
    factories.add(TypeAdapters.STRING_FACTORY); //里边是一些基本类型的adapter啦
       ......
    this.factories = Collections.unmodifiableList(factories);
```
&emsp;Gson内置了很多基本类型和对象的转换组件，类型为`TypeAdapter`,可以通过相应的`TypeAdapterFactory`来获得，所以这里`factories`就预先加载了很多基本类型转换组件的Factory,然后`factories.addAll(typeAdapterFactories)`是添加构造函数中传入的用户自定义的`TypeAdapterFactory`
 &emsp;`fromJson`和`toJson`这两类函数我们在介绍完所有的类之后在解析吧。
#### 3.TypeAdapter
&emsp;这是一个抽象类，提供了两个抽象函数作为hook函数来让用户重载，分别是`public abstract void write(JsonWriter out, T value) throws IOException;`和` public abstract T read(JsonReader in) throws IOException;`一读一写，用户如果要解析自己自定义的对象，就可以继承这个类，然后实现上述两个方法，并在`GsonBuilder中使用`GsonBuilder registerTypeAdapter(Type type, Object typeAdapter) `来注册这个转换类，然后Gson就可以对你的自定义对象进行转换啦。需要注意的是，这里的转换完全由你自己控制，所以可定制性比较强。在介绍`TypeAdapters`时，我们会介绍几个简单的TypeAdapter的实现。
#### 4.TypeAdapterFactory
&emsp;这是一个接口，自定义了一个函数:`<T> TypeAdapter<T> create(Gson gson, TypeToken<T> type);`,具体的实现方法，我们可以在后边的`TypeAdapters`类中看到.
#### 5.TypeAdapters
&emsp;这个类中定义了几乎所有的基础类型的TypeAdapter和Factory,我们现在挑出一个来研究一下。这是一个URL对象的JSON转换器啦。
```
public static final TypeAdapter<URL> URL = new TypeAdapter<URL>() {
    @Override
    public URL read(JsonReader in) throws IOException {
      if (in.peek() == JsonToken.NULL) {
        in.nextNull();
        return null;
      }
      String nextString = in.nextString(); //读出in中的内容
      return "null".equals(nextString) ? null : new URL(nextString);//根据读出的内容，创建URL对象
    }
    @Override
    public void write(JsonWriter out, URL value) throws IOException {
      out.value(value == null ? null : value.toExternalForm()); //写入内容
    }
  };

  public static final TypeAdapterFactory URL_FACTORY = newFactory(URL.class, URL);//创建Facotry啦
```
 &emsp; 我们可以看到，这是一个URL的转换器，`JsonReader`和`JsonWriter`后边会介绍到，现在你就可以把他们当做类似于StringBuilder一类的Json的生成器和解释器。

#### 6.TypeToken
&emsp;TypeToken可以看做是对Java范型的扩展，大家都知道Java范型是有类型擦除效果的，无法获得其真实类型。而这个类就是为了处理这种情况的，我们从文件中的注释也可以了解到。具体的内容，我们希望在Gson相关的第二篇博文中再详细说明，主要就是涉及围绕TypeToken的一系列的Gson对范型的支持和处理。
&emsp;而在Gson对象中，TypeToken主要是用于根据Type来获得相应的TypeAdapter。
#### 其他类
&emsp;还有很多其他的类没有介绍，其中有些不太重要，我也没有太多了解，另外一些我会在接下去的两篇博文中详细介绍。

### 转换流程
&emsp;接下来，我们就主要理通Gson两个最重要的函数的逻辑，之后的两篇博文会详细介绍其中的重要的细节，这篇博文就只诉说每一步大致的作用啦。
#### 1 fromJson()
&emsp;函数名为fromJson的函数比较多，我们只看下边这个
``` 
 @SuppressWarnings("unchecked")
  public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    boolean isEmpty = true;
    boolean oldLenient = reader.isLenient();
    reader.setLenient(true);
    try {
      reader.peek();
      isEmpty = false;
      // 反射部分的精髓,主要的就是TypeToken和TypeAdapter啦
      TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);  //1:工厂方法,其中调用typeToken的构造器
      TypeAdapter<T> typeAdapter = getAdapter(typeToken); //2:通过type来获得Adapter啊
      T object = typeAdapter.read(reader);//3:通过typeAdapter来转换对象
      return object;
    } catch (EOFException e) {
      /*
       * For compatibility with JSON 1.5 and earlier, we return null for empty
       * documents instead of throwing.
       */
      if (isEmpty) {
        return null;
      }
      throw new JsonSyntaxException(e);
    } catch (IllegalStateException e) {
      throw new JsonSyntaxException(e);
    } catch (IOException e) {
      // TODO(inder): Figure out whether it is indeed right to rethrow this as JsonSyntaxException
      throw new JsonSyntaxException(e);
    } finally {
      reader.setLenient(oldLenient);
    }
  }
```
&emsp;如同代码中标注的一样，`fromJson`中大致分为3个比较重要的步奏.

- TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT); 获得要转换类型对应的TypeToken对象，主要涉及的Gson中范型和反射的部分逻辑，我们第二篇博文再讲

- TypeAdapter<T> typeAdapter = getAdapter(typeToken); //通过type来获得Adapter啊,这个我们先来看一下getAdapter函数,就是找出TypeToken所对应的TypeAdapter对象，用于下一步的解析。
```
@SuppressWarnings("unchecked")
  public <T> TypeAdapter<T> getAdapter(TypeToken<T> type) {
    TypeAdapter<?> cached = typeTokenCache.get(type);  // typeTokenCache?? 创造adapter是很麻烦的事情吗？有cache
    if (cached != null) {
      return (TypeAdapter<T>) cached;

    }

    Map<TypeToken<?>, FutureTypeAdapter<?>> threadCalls = calls.get();  //threadLocal get
    boolean requiresThreadLocalCleanup = false; //是否需要清理threadLocal中的数据
    if (threadCalls == null) {
      threadCalls = new HashMap<TypeToken<?>, FutureTypeAdapter<?>>();
      calls.set(threadCalls);
      requiresThreadLocalCleanup = true;
    }

    // the key and value type parameters always agree
    FutureTypeAdapter<T> ongoingCall = (FutureTypeAdapter<T>) threadCalls.get(type);
    if (ongoingCall != null) {
      return ongoingCall;
    }

    try {
      FutureTypeAdapter<T> call = new FutureTypeAdapter<T>();
      threadCalls.put(type, call);

      for (TypeAdapterFactory factory : factories) {
        TypeAdapter<T> candidate = factory.create(this, type);  // 通过factory 来创建TypeAdapter啊，由于需要遍历list比较麻烦
        if (candidate != null) {
          call.setDelegate(candidate);
          typeTokenCache.put(type, candidate);
          return candidate;
        }
      }
     ....
    } finally {
      threadCalls.remove(type);
	  ....
    }
  }
```
 - T object = typeAdapter.read(reader);//3:通过typeAdapter来转换对象,具体过程，和JsonReader，JsonWriter的原理，第三篇博文再进行讲述
#### 2 toJson
&emsp;其实toJson和fromJson很像，就是获得相应的TypeAdapter,只不过这次调用的是write方法。这里就不累赘多说啦。
### 总结
&emsp;博客还未写完，代码还没有看透.....
 
