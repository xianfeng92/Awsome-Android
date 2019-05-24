# Parcelable

Interface for classes whose instances can be written to and restored from a Parcel. Classes implementing the Parcelable interface must also have a non-null 
static field called CREATOR of a type that implements the Parcelable.Creator interface.


一个 Parcelable 的典型实现如下:

```
 public class MyParcelable implements Parcelable {
     private int mData;

     public int describeContents() {
         return 0;
     }

     public void writeToParcel(Parcel out, int flags) {
         out.writeInt(mData);
     }

     public static final Parcelable.Creator<MyParcelable> CREATOR
             = new Parcelable.Creator<MyParcelable>() {
         public MyParcelable createFromParcel(Parcel in) {
             return new MyParcelable(in);
         }

         public MyParcelable[] newArray(int size) {
             return new MyParcelable[size];
         }
     };

     private MyParcelable(Parcel in) {
         mData = in.readInt();
     }
 }
```
其中 describeContents 就是负责文件描述,只针对一些特殊的需要描述信息的对象,需要返回1,其他情况返回0就可以.通过 writeToParcel 方法实现序列化, writeToParcel 返回了Parcel,
所以我们可以直接调用 Parcel 中的 write 方法. CREATOR 负责反序列化,其中 createFromParcel 方法会从序列化后的对象中创建原始对象, newArray 方法用于创建指定长度的原始对象数组


Note: 如果实现Parcelable接口的对象中包含对象或者集合,那么其中的对象也要实现Parcelable接口, 如下是一个案例:

```
public class Book implements Parcelable {

    public int id;
    public String name;
    public boolean isCell;
    public ArrayList<String> tags;
    // Author 必须也要是序列化
    public Author author;

    // ***** 注意: 这里如果是集合 ,一定要初始化 *****
    public ArrayList<Author> authors = new ArrayList<>();

    protected Book(Parcel in) {
        id = in.readInt();
        name = in.readString();
        isCell = in.readByte() != 0;

        tags = in.createStringArrayList();

        author = in.readParcelable(Author.class.getClassLoader());

        in.readList(authors, Author.class.getClassLoader());
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
        dest.writeByte((byte) (isCell? 1: 0));
        // 序列化一个String的集合
        dest.writeStringList(tags);
        // 序列化对象的时候传入要序列化的对象和一个flag,这里的flag几乎都是0
        // 除非标识当前对象需要作为返回值返回,不能立即释放资源
        dest.writeParcelable(author,0);

        // 序列化一个对象的集合
        dest.writeList(authors);
    }
}

```






