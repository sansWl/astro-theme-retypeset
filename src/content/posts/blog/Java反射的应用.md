---
 title: Java反射的应用
 published: 2025-03-03
 tags:
  - BlogPost
 lang: zh
 abbrlink:  f242a294-4b91 
---

> 按照对象属性排序（对象属性需要实现Comparable 【常见数据类型】）
```java
 public static int getOrder(Demo o1, Demo o2, String order) {
        Field field;
        Comparable value1;
        Comparable value2;
        try {
            //查看是否存在排序字段，需要保证排序字段存在且是可比较的，即实现了Comparable接口
            field = Arrays.stream(o1.getClass().getDeclaredFields()).filter(f -> f.getName().equals(order)).findAny().orElse(null);
            if (field != null) {
                // 要求的排序字段不存在，不排序
                return 0;
            }
            field.setAccessible(true);
            value1 = (Comparable) field.get(o1);
            value2 = (Comparable) field.get(o2);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return value1.compareTo(value2);
    }
```
> 类方法的直接调用,声明一个顶级父类（接口），读取继承/实现父类的子实现方法，直接调用
```java
 //定义一个接口，用于定义处理器
    public interface ArrayHandler{

    }
//检查方法签名是否符合要求
  private Function<List, List> checkMethodAndReturnFunction(ArrayHandler arrayHandler,Method method) {
        Type returnType = method.getReturnType();
        Type[] parameterTypes = method.getParameterTypes();
        //check if the method parameter is List && parameter size is 1 and return type is List
        if (parameterTypes.length == 1 && parameterTypes[0] == List.class && returnType == List.class){
           //相当于一个Function，内部是invoke方法调用，当function调用apply时执行
              return list -> {
               try {
                   return (List) method.invoke(arrayHandler,list);
               } catch (IllegalAccessException e) {
                   throw new RuntimeException(e);
               } catch (InvocationTargetException e) {
                   throw new RuntimeException(e);
               }
           };
        }
        log.warn("The method {} is not a valid array handler, please check the method signature.",method.getName());
        return null;
    }

 public  List applyArrayLinkedHandler(List input){
        List output=input;
        synchronized(monitor){
            if(ARRAYHANDLER.isEmpty()){
                log.warn("No array handler found, please add array handler first.");
                return output;
            }
            for(Function<List,List> handler:ARRAYHANDLER){
                output=handler.apply(output);
            }
        }
        return output;
    }

    public  List applyArrayHandler(List input) {
        List output = input;
        Set result;
        synchronized (monitor) {
            if (ARRAYHANDLER.isEmpty()) {
                log.warn("No array handler found, please add array handler first.");
                return output;
            }
            result = new HashSet();
            for (Function<List, List> handler : ARRAYHANDLER) {
                 result.addAll(handler.apply(output));
            }
        }
        return new ArrayList<>(result);
    }
```
> 代理模式(以Jdk代理为例【jdk代理需要类实现接口】)，反射获取对象以及方法的执行
```java
 public static void main(String[] args) {
        Demo helloWorld = new HelloWorldImpl();
        Demo proxy = new JDKProxyDemo(helloWorld).getProxy(Demo.class);
        String s = proxy.sayHello("world");
        System.out.println();
    }

    static class JDKProxyDemo implements InvocationHandler {
        private Demo demo;
        public JDKProxyDemo(Demo demo) {
            this.demo = demo;
        }
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("JDKProxyDemo invoke");
            return method.invoke(demo, args);
        }

        public <T>  T getProxy(Class<T> clazz){
            return (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, this);
        }
    }
```