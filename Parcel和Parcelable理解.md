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

这似乎是一到被
