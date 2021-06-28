---
title: Room原理浅析
tags: jetpack
categories: Android
thumbnail: 'https://dreamweaver.img.we1code.cn/room.jpg'
date: 2019-03-21 00:00:00
top: 0
---

## 前言

> Room是Google推出的数据库处理框架，Jetpack中的一员

## 版本号

```groovy
androidx.room:room-common:2.2.3
androidx.room:room-runtime:2.2.3
androidx.room:room-compiler:2.2.3
```

<!--more-->

## 使用示例

声明一个抽象类继承RoomDatabase，在这个类中主要是写一些getDao的方法，同时也需要用Database注解这个类，告知compiler这个数据库的实体类/版本号等信息，如下示例

```kotlin
@Database(entities=[Fruit::class,Meat::class],version=1)
abstract class AppDataBase:RoomDatabase(){

    abstract fun getFruitDao():FruitDao

    abstract fun getMeatDao():MeatDao
}
```

dao文件的声明如下

```kotlin
@Dao
interface FruitDao{
    @Insert(onConflict=OnConflictStrategy.REPLACE)
    fun insertFruit(fruit:Fruit)

        @Query("DELETE FROM fruit")
        fun deleteFruit()

        @Query("SELECT*FROM fruit")
        fun queryFruit():Fruit
    }
```

好了，通过下面的方法我们就能拿到database对象了，有了RoomDatabase就能操作dao对表做CRUD操作了。

```kotlin
Room.databaseBuilder(application,AppDataBase::class.java,DB_NAME).build()
```

Room的使用流程大概如上，有如下几个疑问：

> 1. RoomTestDataBase是个抽象类，抽象方法在哪实现的？
> 2. RoomTestDataBase内部的Dao是interface，内部的增删改查又是在哪实现的？

### 创建Room Database流程

```java
        @NonNull
        public T build() {
            //noinspection ConstantConditions
            //预检测参数 省略...
            if (mFactory == null) {
                // SQLiteOpenHelper工厂类 创建SQLiteOpenHelper真正操作数据库
                mFactory = new FrameworkSQLiteOpenHelperFactory();
            }
            //DataBase构建参数
            DatabaseConfiguration configuration =
                    new DatabaseConfiguration(mContext, mName, mFactory, mMigrationContainer,
                            mCallbacks, mAllowMainThreadQueries,
                            mJournalMode.resolve(mContext),
                            mRequireMigration, mMigrationsNotRequiredFrom);
            T db = Room.getGeneratedImplementation(mDatabaseClass, DB_IMPL_SUFFIX);
            // 初始化 RoomDataBaseOpenHelper
            db.init(configuration);
            return db;
        }

        // 由方法名可知在这里生成实现类
        @NonNull
        static <T, C> T getGeneratedImplementation(Class<C> klass, String suffix) {
            final String fullPackage = klass.getPackage().getName();
            String name = klass.getCanonicalName();
            final String postPackageName = fullPackage.isEmpty()
                    ? name
                    : (name.substring(fullPackage.length() + 1));
            final String implName = postPackageName.replace('.', '_') + suffix;
            //noinspection TryWithIdenticalCatches
            try {
                @SuppressWarnings("unchecked")
                // 通过反射获取xxxDataBase_Impl，该类通过room-compiler生成
                final Class<T> aClass = (Class<T>) Class.forName(
                        fullPackage.isEmpty() ? implName : fullPackage + "." + implName);
                return aClass.newInstance();
            }
            //省略 catch exceptions
        }
```

> 简单的分析一下上面代码逻辑

我们从Room.databaseBuilder().build()分析room是如何将数据库创建出来的，builder中的变量就是创建database所需的参数，在build()中进行了参数的预校验，最终调用Room.getGenratedImplementation()，拼接出类名AppDataBase_Imp，通过反射创建对象，而这个AppDataBase_Imp则是通过注解处理器自动生成的，它继承了抽象类AppDataBase，再来趴一下AppDataBase_Imp，在createOpenHelper函数中，它利用SupportSQLiteOpenHelper.Factory工厂类创建了SupportSQLiteOpenHelper，这个工厂类在建造者模式Room.databaseBuilder().build()中被初始化，真正调用的是FrameworkSQLiteOpenHelperFactory.create函数，创建了FrameworkSQLiteOpenHelper，在RoomDatabase使用的databaseOpenHelper也就是这个类了，其实FrameworkSQLiteOpenHelper是个代理类，它内部的OpenHelper继承了SQLiteOpenHelper，到此终于看到了熟悉的类型，我们直接使用SQLite通常也是直接写一个helper类直接继承SQLiteOpenHelper，而ROOM通过层层装饰省去了我们创建SQLiteOpenHelper的麻烦，最后用一张流程图总结ROOM创建database的过程。

![room创建流程图](https://dreamweaver.img.we1code.cn/room.png)

### 指定Room schema的输出路径

在我们使用room Database注解时，会发现有exportSchema属性，官方注释如下：

```java
/**
     * You can set the annotation processor argument ({@code room.schemaLocation}) to tell Room to
     * export the database schema into a folder. Even though it is not mandatory, it is a good
     * practice to have version history of your schema in your codebase and you should commit the
     * schema files into your version control system (but don't ship them with your app!).
     * <p>
     * When {@code room.schemaLocation} is set, Room will check this variable and if it is set to
     * {@code true}, the database schema will be exported into the given folder.
     * <p>
     * {@code exportSchema} is {@code true} by default but you can disable it for databases when
     * you don't want to keep history of versions (like an in-memory only database).
     *
     * @return Whether the schema should be exported to the given folder when the
     * {@code room.schemaLocation} argument is set. Defaults to {@code true}.
     */
    boolean exportSchema() default true;
```

大概是说这个属性是控制是否输出schema的，建议大家将数据库的schema做版本控制，方便数据库的迁移和版本升级/降级。

在gradle可以配置schema的输出目录，具体代码如下

```groovy
android{
    ...//省略

    defaultConfig {
        ... //省略

        javaCompileOptions{
            annotationProcessorOptions{
                arguments=["room.schemaLocation":"$projectDir/schemas".toString()]
            }
        }
    }
}
```

### Room的版本迁移

在开发过程中数据库的表结构可能会发生变化，在Room中如果你不做相应的处理，则会抛出异常并提示你A migration from old_version to new_version was required but not found，
我们可以通过addMigrations()处理版本迁移中数据库的变动，也可以通fallbackToDestructiveMigration/fallbackToDestructiveMigrationOnDowngrade/fallbackToDestructiveMigrationFrom
设置数据库在版本升级的时候，删除原表并创建新表，但是会导致原数据的丢失，因此，为了防止用户数据丢失，我们需要管理好自己数据库的版本，并在版本升级时处理好变动的数据。下面示范了Fruit表中新增字段后是如何做处理的

```kotlin
valversion1To2=object:Migration(1,2){
    override fun migrate(database:SupportSQLiteDatabase){
        database.execSQL("ALTER TABLE fruit ADD COLUMN test TEXT NOT NULL DEFAULT''")
    }
}
```

当遇到比较复杂的情况时，只要我们做好数据库的版本管理就都能很好的解决。

### CRUD流程

接下来看看ROOM的CRUD流程，我们可以从DAO入手，dao_Impl也是编译时生成的文件，看看FruitDao_Impl的insertFruit函数，如下所示

```java
public void insertFruit(final Fruit fruit){
    __db.assertNotSuspendingTransaction();
    __db.beginTransaction();
    try{
        __insertionAdapterOfFruit.insert(fruit);
        __db.setTransactionSuccessful();
    }finally{
        __db.endTransaction();
    }
}
```

它对应的FruitDao函数

```kotlin
@Insert(onConflict = OnConflictStrategy.REPLACE)
fun insertFruit(fruit: Fruit)
```

插入逻辑

```kotlin
public final void insert(T entity) {
    final SupportSQLiteStatement stmt = acquire();
    try {
        bind(stmt, entity); // 绑定数据
        stmt.executeInsert(); // 执行插入语句
    } finally {
        release(stmt);
    }
}
```

最终将调用的函数

```java
    // SQLiteProgram
    private void bind(int index, Object value) {
        // ...省略
        mBindArgs[index - 1] = value; // 将数据存入
    }

    // SQLiteStatement
    public long executeInsert() {
        acquireReference();
        try {
            return getSession().executeForLastInsertedRowId(
                    getSql(), getBindArgs(), getConnectionFlags(), null);
        } catch (SQLiteDatabaseCorruptException ex) {
            onCorruption();
            throw ex;
        } finally {
            releaseReference();
        }
    }
```

## 结语

关于Room的原理和基本使用方法就介绍到这了，本来还想分析一下room-compiler的，但是感觉写下去篇幅会太长了。下节将借room-compiler总结一下annotation-processor的开发流程。
