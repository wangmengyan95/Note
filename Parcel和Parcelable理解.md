#Parcel 
Parcel是android跨进程通信的基础。Parcel存储了需要在进程间分享的信息，并且提供了读取和存储信息的接口。

Parcel的源代码在[这里](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/Parcel.java)。

Parcel支持存取的数据类型包括
* Primitives 包括byte, doulble ...
* Primitive 数组，包括boolean数组， byte数组，double数组 ...
* FileDescrpotr
* Binder
* Parcelable，即实现了Parcelable接口的对象
* Serializable，即实现了Serializable接口的对象

#Parcelable
Parcelable接口定义了可以被Parcel存取传输的对象需要实现的方法。一个简单的Parcelable对象的例子如下
```
private static class People implements Parcelable {

    private int mAge;
    private String mName;

    public People(int age, String name) {
        mAge = age;
        mName = name;
    }

    protected People(Parcel in) {
        mAge = in.readInt();
        mName = in.readString();
    }

    public static final Creator<People> CREATOR = new Creator<People>() {
        @Override
        public People createFromParcel(Parcel in) {
            return new People(in);
        }

        @Override
        public People[] newArray(int size) {
            return new People[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeInt(mAge);
        parcel.writeString(mName);
    }
}
```

* writeToParcel(...)方法是控制向Parcel里写入数据的方法，在这里我们只要将需要Parcel传输的数据依次调用相信的方法写入Parcel即可。
* CREATOR是一个static field，根据Parceable的文档，所有Parcelable对象都需要有这个field，它主要负责如何从Parcel里读取数据重新构建对象。特别需要注意的是
createFromParcel里从Parcel读取数据的顺序应该和writeToParcel向Parcel写入数据的顺序一致。
* describeContents()方法也是Parcelable对象必须实现的方法。关于这个方法的含义官方文档我觉得并没有说清楚，我从google group[这里](https://groups.google.com/forum/#!topic/android-developers/QVSWGS-8W0A)找到了一个相对更好理解的答案。简单点说，Bundle.hasFileDescriptors()会依赖于这个方法的返回值判断Parcel里是否含有FileDescriptor。一般来说如果不传输FileDescriptor，返回0即可。如果传输FileDescriptor，像ParcelableFileDescriptor一样返回Parcelable.CONTENTS_FILE_DESCRIPTOR即可。

#Parcelable vs Serializable

这似乎是一个被经常问到的面试题，我们可以从Parcel的源码中看看两者到底在使用的时候有什么区别。

##Parcelable
```
public final void writeParcelable(Parcelable p, int parcelableFlags) {
    if (p == null) {
        writeString(null);
        return;
    }
    writeParcelableCreator(p);
    p.writeToParcel(this, parcelableFlags);
}

public final void writeParcelableCreator(Parcelable p) {
    String name = p.getClass().getName();
    writeString(name);
}
```
将Parcelable写入Parcel的方法非常简单，我们首先向Parcel中写入Parcelable对象的实际类名，接着调用Parcelable.writeToParcel()写入具体数据即可。


```
public final <T extends Parcelable> T readParcelable(ClassLoader loader) {
    Parcelable.Creator<?> creator = readParcelableCreator(loader);
    if (creator == null) {
        return null;
    }
    if (creator instanceof Parcelable.ClassLoaderCreator<?>) {
      Parcelable.ClassLoaderCreator<?> classLoaderCreator =
          (Parcelable.ClassLoaderCreator<?>) creator;
      return (T) classLoaderCreator.createFromParcel(this, loader);
    }
    return (T) creator.createFromParcel(this);
}

public final Parcelable.Creator<?> readParcelableCreator(ClassLoader loader) {
    String name = readString();
    if (name == null) {
        return null;
    }
    Parcelable.Creator<?> creator;
    synchronized (mCreators) {
        HashMap<String,Parcelable.Creator<?>> map = mCreators.get(loader);
        if (map == null) {
            map = new HashMap<>();
            mCreators.put(loader, map);
        }
        creator = map.get(name);
        if (creator == null) {
            try {
                // If loader == null, explicitly emulate Class.forName(String) "caller
                // classloader" behavior.
                ClassLoader parcelableClassLoader =
                        (loader == null ? getClass().getClassLoader() : loader);
                // Avoid initializing the Parcelable class until we know it implements
                // Parcelable and has the necessary CREATOR field. http://b/1171613.
                Class<?> parcelableClass = Class.forName(name, false /* initialize */,
                        parcelableClassLoader);
                if (!Parcelable.class.isAssignableFrom(parcelableClass)) {
                    throw new BadParcelableException("Parcelable protocol requires that the "
                            + "class implements Parcelable");
                }
                Field f = parcelableClass.getField("CREATOR");
                if ((f.getModifiers() & Modifier.STATIC) == 0) {
                    throw new BadParcelableException("Parcelable protocol requires "
                            + "the CREATOR object to be static on class " + name);
                }
                Class<?> creatorType = f.getType();
                if (!Parcelable.Creator.class.isAssignableFrom(creatorType)) {
                    // Fail before calling Field.get(), not after, to avoid initializing
                    // parcelableClass unnecessarily.
                    throw new BadParcelableException("Parcelable protocol requires a "
                            + "Parcelable.Creator object called "
                            + "CREATOR on class " + name);
                }
                creator = (Parcelable.Creator<?>) f.get(null);
            }
            
            ...
            
            map.put(name, creator);
        }
    }

    return creator;
}
```
从Parcel中读取Parcelable的过程略显复杂
* 我们首先需要用readParcelableCreator()获得Parcelable对象的CREATOR，Parcel维护了一个内部的缓存mCreators方便我们找到对应的CREATOR，当时当我们第一次尝试获得CREATOR时，我们需要通过反射创建Parcelable对象的CREATOR。
* 当取得Parcelable对象的CREATOR后，我们通过调用自定义的createFromParcel(..)重建出对象。

##Serializable
Parcel中对Serializable对象的读取利用了Java本身的ObjectInputStream，ObjectOutputStream。Parcel利用ObjectInputStream，ObjectOutputStream完成了对Serializable对象的序列化和反序列化。

对于ObjectOutputStream Serializable对象的写入，它最核心就是writeSerialData()这个方法。在这个方法中，会首先判断对象是否重载了writeObject(...)方法（`slotDesc.hasWriteObjectMethod()`），如果有则调用（`slotDesc.invokeWriteObject(obj, this)`），否则就调用默认的写入方法（`defaultWriteFields(obj, slotDesc)`）。对于
