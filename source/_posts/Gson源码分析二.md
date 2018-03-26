title: Gson源码分析二
date: 2015-10-07 15:48:51
tags: 
	- 第三方库
	- 反射
	- JSON
categories:
	- Java
---

&emsp;承接上一篇博文[Gson源码分析](http://blog.csdn.net/u012422440/article/details/48860893)，这篇博文主要总结一下Gson中涉及Java反射逻辑的部分。
#### 一个Gson例子
&emsp;Gson可以解析用户自定义的对象，当然你也可以使用`public GsonBuilder registerTypeAdapter(Type type, Object typeAdapter) `来完全按照自己的方式来解析，但是Gson其实已经为解析自定义类型对象做了适配，除非特殊需求，我们一般不需要定义自己的TypeAdapter。比如下边这个例子
```
class MyType{
    private int i = 1;
    private String name = "test";

    public MyType() {
        super();
    }
    public MyType(int id,String name) {
        this();
    }
}
.....
 MyType type = new MyType();
 System.out.println(gson.toJson(type));
        
```
&emsp;这样就可以将MyType对象与JSON格式字符串进行相互转换了，不得不说这是十分方便的。而且通过`Excluder`和`FieldNamingStragety`我们还可以对Gson的转换过程进行一定的控制。
&emsp;更为厉害的是，Gson对相对泛型和复杂的对象支持的很好，比如`ArrayList<MyType>`对象，也可以直接通过Gson进行转换。
### Gson反射基础
&emsp;这一部分主要讲解一下Java相关的反射基础和Gson对其进行的扩展。
&emsp;我们都知道Java泛型是类型擦除的，也就是说在运行期间，我们无法通过反射获得泛型对象的类型，那Gson是如何解析类似于`ArrayList<MyType>`这样的对象的类型的呢？
&emsp;上篇博文中讲到了Gson中使用TypeToken来代表对象的类型，其创建时会使用到`$Gson$Types`这个对象，我们现在就来好好研究一下这个对象。
```
public static Type canonicalize(Type type) {
    if (type instanceof Class) {  //如果就是Class
      Class<?> c = (Class<?>) type;
        // c.getComponetType()就是返回数组对象的类型比如
      return c.isArray() ? new GenericArrayTypeImpl(canonicalize(c.getComponentType())) : c;
      // ????!!!! loop for ever ????
    } else if (type instanceof ParameterizedType) {  //多级泛型 HashMap<K,T>
      ParameterizedType p = (ParameterizedType) type;
      return new ParameterizedTypeImpl(p.getOwnerType(),
          p.getRawType(), p.getActualTypeArguments());
    } else if (type instanceof GenericArrayType) {   //数组泛型
      GenericArrayType g = (GenericArrayType) type;
      return new GenericArrayTypeImpl(g.getGenericComponentType());
    } else if (type instanceof WildcardType) {   // includes ?  , ? extends Number , ? super T
      WildcardType w = (WildcardType) type;
      return new WildcardTypeImpl(w.getUpperBounds(), w.getLowerBounds());
    } else {
      // type is either serializable as-is or unsupported
      return type;
    }
  }
```

&emsp;这里的`ParameterizedType`,`GenericArrayType`,`WildcardType`还有之后会出现的`TypeVariable`是一个重点啊，他们都是`Type`的子接口，代表所有类型的公共高阶接口。详细的解释在这里有[转载博文](http://blog.csdn.net/u012422440/article/details/48948921)
&emsp;还需要进一步的实验啊，以后再来补充这一部分。还有关于如何处理泛型类型擦除的逻辑。
###ReflectiveTypeAdapterFacotry
&emsp;在上边博文中我们说过，Gson会使用`TypeToken`来代表转换对象的类型，然后找到对应类型的`TypeAdapter`，但是对于用户自定义的类型，Gson是如何处理的呢？
&emsp;Gson的`TypeAdapters`中有一个可以处理自定义类型的`TypeAdapterFactory`，它就是`ReflectiveTypeAdapterFactory`,它也是这篇博文的重点内容。
```
 private final ConstructorConstructor constructorConstructor; //构造函数
  private final FieldNamingStrategy fieldNamingPolicy; //命名规则
  private final Excluder excluder; //排除器
```
&emsp;上述是其成员变量，文章开头所说的用户控制Gson的转换过程就是通过这三个对象实现的，我接下来会一一讲解。
&emsp;我们知道在Gson中通过`TypeToken`获得相应的`TypeAdapter`的逻辑如下
```
 for (TypeAdapterFactory factory : factories) {
        TypeAdapter<T> candidate = factory.create(this, type);
          System.out.print("1 ");
        if (candidate != null) {
            System.out.println(candidate.toString()+type.toString());
          call.setDelegate(candidate);
          typeTokenCache.put(type, candidate);
          return candidate;
        }
      }
```
&emsp;可以看出只要对应的facotry的create返回的对象不为null，就认为找到了对应的factory了，那我们在来看一下`ReflectiveTypeAdapterFactory`的create函数
```
public <T> TypeAdapter<T> create(Gson gson, final TypeToken<T> type) {
    Class<? super T> raw = type.getRawType();
    if (!Object.class.isAssignableFrom(raw)) {  
    //如果Object都不是raw的最高类型,表示raw不是Object的子类啦 
	 return null; // it's a primitive!
    }
    ObjectConstructor<T> constructor = constructorConstructor.get(type);
    return new Adapter<T>(constructor, getBoundFields(gson, type, raw));
  }
```
&emsp;ObjectConstrutor<T>是为了构造器，为了创建一个相应的对象，主要是fromJson时使用的,而getBoundFields是为了获得对象的Filed类型。
```
    final InstanceCreator<T> typeCreator = (InstanceCreator<T>) instanceCreators.get(type); //no-args和type一一对应啊
    if (typeCreator != null) {
      return new ObjectConstructor<T>() {  //这就是最基础的Constructor啊
        public T construct() {
          return typeCreator.createInstance(type);
        }
      };
    }

    // Next try raw type match for instance creators
    @SuppressWarnings("unchecked") // types must agree
    final InstanceCreator<T> rawTypeCreator =
        (InstanceCreator<T>) instanceCreators.get(rawType);
    if (rawTypeCreator != null) {
      return new ObjectConstructor<T>() {
        public T construct() {
          return rawTypeCreator.createInstance(type);
        }
      };
    }

    ObjectConstructor<T> defaultConstructor = newDefaultConstructor(rawType); //默认的构造函数
    if (defaultConstructor != null) {
      return defaultConstructor;
    }


    ObjectConstructor<T> defaultImplementation = newDefaultImplementationConstructor(type, rawType);
    if (defaultImplementation != null) {
      return defaultImplementation;
    }

    // finally try unsafe
    return newUnsafeAllocator(type, rawType);
  }
```
&emsp;这段代码就是为了获得对应类型的构造器对象，在`newDefaultConstructor`会使用反射`getDeclaredConstructor`来获得默认的构造器。然后我在来看一下`getBoundFields`
```
 //获得成员变量吧
  private Map<String, BoundField> getBoundFields(Gson context, TypeToken<?> type, Class<?> raw) {
    Map<String, BoundField> result = new LinkedHashMap<String, BoundField>();
    if (raw.isInterface()) { //interface没有field啦
      return result;
    }

    Type declaredType = type.getType();
    while (raw != Object.class) { //从当前类型一直遍历到最高类型，把所有的对象的成员遍历都收集到
      Field[] fields = raw.getDeclaredFields();
      for (Field field : fields) {  //遍历所有的field
        boolean serialize = excludeField(field, true);//是否需要序列化，就是是否需要转换成Json
        boolean deserialize = excludeField(field, false);//是否需要从Json中转换过来
        if (!serialize && !deserialize) {
          continue;
        }
        field.setAccessible(true);
          //获得Field的type
        Type fieldType = $Gson$Types.resolve(type.getType(), raw, field.getGenericType());//获得Field的Type
        BoundField boundField = createBoundField(context, field, getFieldName(field),
            TypeToken.get(fieldType), serialize, deserialize); //之后详细解释
        BoundField previous = result.put(boundField.name, boundField);
        if (previous != null) {
          throw new IllegalArgumentException(declaredType
              + " declares multiple JSON fields named " + previous.name);
        }
      }
      type = TypeToken.get($Gson$Types.resolve(type.getType(), raw, raw.getGenericSuperclass()));
      raw = type.getRawType();
    }
    return result;
  }
   static abstract class BoundField {
    final String name;
    final boolean serialized;
    final boolean deserialized;

    protected BoundField(String name, boolean serialized, boolean deserialized) {
      this.name = name;
      this.serialized = serialized;
      this.deserialized = deserialized;
    }
```
&emsp;其实这一段代码逻辑很简单，主要就是遍历所有的成员遍历，大家可以看我的注释，其中涉及`Type`的操作我还没有搞懂....,但是`createBoundField`也是很重要的。
```
private ReflectiveTypeAdapterFactory.BoundField createBoundField(
      final Gson context, final Field field, final String name,
      final TypeToken<?> fieldType, boolean serialize, boolean deserialize) {
    final boolean isPrimitive = Primitives.isPrimitive(fieldType.getRawType());
    // special casing primitives here saves ~5% on Android...
    //其实就是创建一个BoundField的子类
    return new ReflectiveTypeAdapterFactory.BoundField(name, serialize, deserialize) {
      final TypeAdapter<?> typeAdapter = getFieldAdapter(context, field, fieldType);//获得子类的TypeAdapter<?>这是解析自定义类型对象的关键一步
      @SuppressWarnings({"unchecked", "rawtypes"}) // the type adapter and field type always agree
      @Override void write(JsonWriter writer, Object value)
          throws IOException, IllegalAccessException {
        Object fieldValue = field.get(value);
        TypeAdapter t =
          new TypeAdapterRuntimeTypeWrapper(context, this.typeAdapter, fieldType.getType());
        t.write(writer, fieldValue); //使用TypeAdapter进行写入
      }
      @Override void read(JsonReader reader, Object value)
          throws IOException, IllegalAccessException {
        Object fieldValue = typeAdapter.read(reader);
        if (fieldValue != null || !isPrimitive) {
          field.set(value, fieldValue);
        }
      }
      public boolean writeField(Object value) throws IOException, IllegalAccessException {
        if (!serialized) return false;
        Object fieldValue = field.get(value);
        return fieldValue != value; // avoid recursion for example for Throwable.cause
      }
    };
  }
```

&emsp;接下来我们就来看一下对应的TypeAdapter `private Adapter(ObjectConstructor<T> constructor, Map<String, BoundField> boundFields)`，主要看其read和write方法。其中两个方法最后其实都是调用了BoundField的read和write方法，我们就只看read方法啦。
```
 @Override public T read(JsonReader in) throws IOException {
      if (in.peek() == JsonToken.NULL) {
        in.nextNull();
        return null;
      }

      T instance = constructor.construct(); //新建对象

      try {
        in.beginObject();//读出一个{
        while (in.hasNext()) {
          String name = in.nextName();//获得一个属性的name
          BoundField field = boundFields.get(name);//获得Field对象
          if (field == null || !field.deserialized) {//如果为null或者不需要解序列化
            in.skipValue();
          } else {
            field.read(in, instance);//使用BoundField进行write，可以参考createBoundField
          }
        }
      } catch (IllegalStateException e) {
        throw new JsonSyntaxException(e);
      } catch (IllegalAccessException e) {
        throw new AssertionError(e);
      }
      in.endObject();
      return instance;
    }
```
&emsp;这样对已自定义对象就可以自由的和JSON格式进行相互转换啦。但是有些同学可能会问了`ArrayList<MyType>`是如何转换的呢？那我们就要研究`CollectionTypeAdapterFactory`了，如果你去internal/bind文件夹下查看，你会发现很多类似的类。下边就贴出来其create函数
```
 public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> typeToken) {
    Type type = typeToken.getType();

    Class<? super T> rawType = typeToken.getRawType();
    if (!Collection.class.isAssignableFrom(rawType))//看是否是Collection的子类 {
      return null;
    }

    Type elementType = $Gson$Types.getCollectionElementType(type, rawType);//获得element的Type就是List<T>的T的类型，同学们可能会疑问泛型不是类型擦除了吗？你可以阅读一下$Gson$Types中的代码，自行了解一下，我暂时还没有完全搞懂
    //之后就和ReflectiveTypeAdapterFactory的逻辑类似啦
    TypeAdapter<?> elementTypeAdapter = gson.getAdapter(TypeToken.get(elementType));
    ObjectConstructor<T> constructor = constructorConstructor.get(typeToken);

    @SuppressWarnings({"unchecked", "rawtypes"}) // create() doesn't define a type parameter
    TypeAdapter<T> result = new Adapter(gson, elementType, elementTypeAdapter, constructor);
    return result;
  }
```
&emsp;感觉关于`TypeToken`和`$Gson$Type`还是没有完全明白，所以并没有过多涉及，大家如果发现神马问题，或者有好的建议，欢迎大家批判和指教。
