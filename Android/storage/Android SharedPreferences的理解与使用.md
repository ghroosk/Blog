**Android 五种数据存储的方式分别为：**
1. SharedPreferences：以Map形式存放简单的配置参数；
2. ContentProvider：将应用的私有数据提供给其他应用使用；
3. 文件存储：以IO流形式存放，可分为手机内部和手机外部（sd卡等）存储，可存放较大数据；
4. SQLite：轻量级、跨平台数据库，将所有数据都是存放在手机上的单一文件内，占用内存小；
5. 网络存储 ：数据存储在服务器上，通过连接网络获取数据；


Sharedpreferences是Android平台上一个轻量级的存储类，用来保存应用程序的各种配置信息，其本质是一个以“键-值”对的方式保存数据的xml文件，其文件保存在/data/data/<package name>/shared_prefs目录下。在全局变量上看，其优点是不会产生Application 、 静态变量的OOM（out of memory）和空指针问题，其缺点是效率没有上面的两种方法高。    
#1.获取SharedPreferences
 要想使用 SharedPreferences 来存储数据，首先需要获取到 SharedPreferences 对象。Android中主要提供了三种方法用于得到 SharedPreferences 对象。
 1. Context 类中的 getSharedPreferences()方法：
 此方法接收两个参数，第一个参数用于指定 SharedPreferences 文件的名称，如果指定的文件不存在则会创建一个，第二个参数用于指定操作模式，主要有以下几种模式可以选择。MODE_PRIVATE 是默认的操作模式，和直接传入 0 效果是相同的。
 MODE_WORLD_READABLE 和 MODE_WORLD_WRITEABLE 这两种模式已在 Android 4.2 版本中被废弃。
```
Context.MODE_PRIVATE: 指定该SharedPreferences数据只能被本应用程序读、写；
Context.MODE_WORLD_READABLE:  指定该SharedPreferences数据能被其他应用程序读，但不能写；
Context.MODE_WORLD_WRITEABLE:  指定该SharedPreferences数据能被其他应用程序读；
Context.MODE_APPEND：该模式会检查文件是否存在，存在就往文件追加内容，否则就创建新文件；
```
 2. Activity 类中的 getPreferences()方法：
 这个方法和 Context 中的 getSharedPreferences()方法很相似，不过它只接收一个操作模式参数，因为使用这个方法时会自动将当前活动的类名作为 SharedPreferences 的文件名。


 3. PreferenceManager 类中的 getDefaultSharedPreferences()方法：
 这是一个静态方法，它接收一个 Context 参数，并自动使用当前应用程序的包名作为前缀来命名 SharedPreferences 文件。


#2.SharedPreferences的使用
SharedPreferences对象本身只能获取数据而不支持存储和修改,存储修改是通过SharedPreferences.edit()获取的内部接口Editor对象实现。使用Preference来存取数据，用到了SharedPreferences接口和SharedPreferences的一个内部接口SharedPreferences.Editor，这两个接口在android.content包中；
```
 1）写入数据：
     //步骤1：创建一个SharedPreferences对象
     SharedPreferences sharedPreferences= getSharedPreferences("data",Context.MODE_PRIVATE);
     //步骤2： 实例化SharedPreferences.Editor对象
     SharedPreferences.Editor editor = sharedPreferences.edit();
     //步骤3：将获取过来的值放入文件
     editor.putString("name", “Tom”);
     editor.putInt("age", 28);
     editor.putBoolean("marrid",false);
     //步骤4：提交               
     editor.commit();


 2）读取数据：
     SharedPreferences sharedPreferences= getSharedPreferences("data", Context .MODE_PRIVATE);
     String userId=sharedPreferences.getString("name","");
  
3）删除指定数据
     editor.remove("name");
     editor.commit();


4）清空数据
     editor.clear();
     editor.commit();
```
    
**注意**：如果在 Fragment 中使用SharedPreferences 时，需要放在onAttach(Activity activity)里面进行SharedPreferences的初始化，否则会报空指针 即 getActivity()会可能返回null ！


 读写其他应用的SharedPreferences 步骤如下（未实践）:
 1. 在创建SharedPreferences时，指定MODE_WORLD_READABLE模式，表明该SharedPreferences数据可以被其他程序读取；
 2. 创建其他应用程序对应的Context；
 3. 使用其他程序的Context获取对应的SharedPreferences；
 4. 如果是写入数据，使用Editor接口即可，所有其他操作均和前面一致；
```
try {
//这里的com.example.mpreferences 就是应用的包名
 Context mcontext = createPackageContext("com.example.mpreferences", CONTEXT_IGNORE_SECURITY);


 SharedPreferences msharedpreferences = mcontext.getSharedPreferences("name_preference", MODE_PRIVATE);
 int count = msharedpreferences.getInt("count", 0);


 } catch (PackageManager.NameNotFoundException e) {
       e.printStackTrace();
 }
```
#3. SharedPreferences 的源码分析（API 25）
先从Context的getSharedPreferences开始：
```
  public abstract SharedPreferences getSharedPreferences(String name, int mode);
```
我们知道Android中的Context类其实是使用了装饰者模式，而被装饰对象其实就是一个ContextImpl对象，ContextImpl的getSharedPreferences方法：
```
    /**
     * Map from preference name to generated path.
     * 从preference名称到生成路径的映射；
     */
    @GuardedBy("ContextImpl.class")
    private ArrayMap<String, File> mSharedPrefsPaths;


    /**
     * Map from package name, to preference name, to cached preferences.
     * 从包名映射到preferences，以缓存preferences，这是个静态变量；
     */
    @GuardedBy("ContextImpl.class")
    private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;


    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }


        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }




    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        checkMode(mode);
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }


    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }


        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }


        return packagePrefs;
    }
```
  从上面我们可以看出他们之间的关系：
mSharedPrefsPaths存放的是名称与文件夹的映射（ArrayMap<String, File>），这里的名称就是我们使用getSharedPreferences时传入的name，如果mSharedPrefsPaths为null则初始化，如果file为null则新建一个File并将其加入mSharedPrefsPaths中；


（ ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> ）
sSharedPrefsCache 存放包名与ArrayMap键值对
初始化时会默认以包名作为键值对中的Key，注意这是个static变量；


（ArrayMap<File, SharedPreferencesImpl>）sSharedPrefs
packagePrefs存放文件name与SharedPreferencesImpl键值对




![image.png](https://upload-images.jianshu.io/upload_images/3064872-cecd61e9cc1c2af0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**注意：**
1. 对于一个相同的SharedPreferences name，获取到的都是同一个SharedPreferences对象，它其实是SharedPreferencesImpl对象。
2. sSharedPrefs在程序中是静态的，如果退出了程序但Context没有被清掉，那么下次进入程序仍然可能取到本应被删除掉的值。而换了另一种清除SharedPreferences的方式：使用SharedPreferences.Editor的commit方法能够起作用，调用后不退出程序都马上生效。


**SharedPreferencesImpl对象**
```
  SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        //  备份的File
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }


   private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                loadFromDisk();
            }
        }.start();
    }


    private void loadFromDisk() {
        synchronized (SharedPreferencesImpl.this) {
            if (mLoaded) {
                return;
            }
            if (mBackupFile.exists()) {
                mFile.delete();
                mBackupFile.renameTo(mFile);
            }
        }


        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }


        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (XmlPullParserException | IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
            /* ignore */
        }


        synchronized (SharedPreferencesImpl.this) {
            mLoaded = true;
            if (map != null) {
                mMap = map;
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
            } else {
                mMap = new HashMap<>();
            }
            notifyAll();
        }
    }


```
可以看到对于一个SharedPreferences文件name，第一次调用getSharedPreferences时会去创建一个SharedPreferencesImpl对象，它会开启一个子线程，然后去把指定的SharedPreferences文件中的键值对全部读取出来，存放在一个Map中。


调用getString时那个SharedPreferencesImpl构造方法开启的子线程可能还没执行完（比如文件比较大时全部读取会比较久），这时getString当然还不能获取到相应的值，必须阻塞到那个子线程读取完为止，如getString方法：
```
    @Nullable
    public String getString(String key, @Nullable String defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }


   private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```
显然这个awaitLoadedLocked方法就是用来等this这个锁的，在loadFromDiskLocked方法的最后我们也可以看到它调用了notifyAll方法，这时如果getString之前阻塞了就会被唤醒。那么这里会存在一个问题，我们的getString是写在UI线程中，如果那个getString被阻塞太久了，比如60s，这时就会出现ANR，所以要根据具体情况考虑是否需要把SharedPreferences的读写放在子线程中。


关于mBackupFile，SharedPreferences在写入时会先把之前的xml文件改成名成一个备份文件，然后再将要写入的数据写到一个新的文件中，如果这个过程执行成功的话，就会把备份文件删除。由此可见每次即使只是添加一个键值对，也会重新写入整个文件的数据，这也说明SharedPreferences只适合保存少量数据，文件太大会有性能问题。




**注意：**
1. 在UI线程中调用getXXX可能会导致ANR。
2. 我们在初始化SharedPreferencesImpl对象时会加SharedPreferencesImpl对应的xml文件中的所有数据都加载到内存中，如果xml文件很大，将会占用大量的内存，我们只想读取xml文件中某个key的值，但我们获取它的时候是会加载整个文件。
3. 每添加一个键值对，都会重新写入整个文件的数据，不是增量写入；
综上原因能说明Sharedpreferences只适合做轻量级的存储。


**SharedPreferences的内部类Editor**


```
   SharedPreferences sharedPreferences= getSharedPreferences("data",Context.MODE_PRIVATE);
   SharedPreferences.Editor editor = sharedPreferences.edit();
   editor.putString("name", “Tom”);


   public Editor edit() {
        // TODO: remove the need to call awaitLoadedLocked() when
        // requesting an editor.  will require some work on the
        // Editor, but then we should be able to do:
        //
        //      context.getSharedPreferences(..).edit().putString(..).apply()
        //
        // ... all without blocking.
        synchronized (this) {
            awaitLoadedLocked();
        }


        return new EditorImpl();
    }
```
其实拿到的是一个EditorImpl对象，它是SharedPreferencesImpl的内部类：
```
  public final class EditorImpl implements Editor {
        private final Map<String, Object> mModified = Maps.newHashMap();
        private boolean mClear = false;


        public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
          .....
}
```
可以看到它有一个Map对象mModified，用来保存“修改的数据”，也就是你每次put的时候其实只是把那个键值对放到这个mModified 中，最后调用apply或者commit才会真正把数据写入文件中，如上面的putString方法，其它putXXX代码基本也是一样的。


**commit方法和apply方法的不同**
```
    public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };


            QueuedWork.add(awaitCommit);


            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };


            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);


            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }


    public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }


    // Return value from EditorImpl#commitToMemory()
    private static class MemoryCommitResult {
        public boolean changesMade;  // any keys different?
        public List<String> keysModified;  // may be null
        public Set<OnSharedPreferenceChangeListener> listeners;  // may be null
        public Map<?, ?> mapToWriteToDisk;
        public final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);
        public volatile boolean writeToDiskResult = false;


        public void setDiskWriteResult(boolean result) {
            writeToDiskResult = result;
            writtenToDiskLatch.countDown();
        }
    }
```
两种方式首先都会先使用commitTomemory函数将修改的内容写入到SharedPreferencesImpl当中，再调用enqueueDiskWrite写磁盘操作，commitToMemory就是产生一个“合适”的MemoryCommitResult对象mcr，然后调用enqueueDiskWrite时需要把这个对象传进去，commitToMemory方法：
```
        // Returns true if any changes were made
        private MemoryCommitResult commitToMemory() {
            MemoryCommitResult mcr = new MemoryCommitResult();
            synchronized (SharedPreferencesImpl.this) {
                // We optimistically don't make a deep copy until
                // a memory commit comes in when we're already
                // writing to disk.
                if (mDiskWritesInFlight > 0) {
                    // We can't modify our mMap as a currently
                    // in-flight write owns it.  Clone it before
                    // modifying it.
                    // noinspection unchecked
                    mMap = new HashMap<String, Object>(mMap);
                }
                mcr.mapToWriteToDisk = mMap;
                mDiskWritesInFlight++;


                boolean hasListeners = mListeners.size() > 0;
                if (hasListeners) {
                    mcr.keysModified = new ArrayList<String>();
                    mcr.listeners =
                            new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
                }


                synchronized (this) {
                    if (mClear) {
                        if (!mMap.isEmpty()) {
                            mcr.changesMade = true;
                            mMap.clear();
                        }
                        mClear = false;
                    }


                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        if (v == this || v == null) {
                            if (!mMap.containsKey(k)) {
                                continue;
                            }
                            mMap.remove(k);
                        } else {
                            if (mMap.containsKey(k)) {
                                Object existingValue = mMap.get(k);
                                if (existingValue != null && existingValue.equals(v)) {
                                    continue;
                                }
                            }
                            mMap.put(k, v);
                        }


                        mcr.changesMade = true;
                        if (hasListeners) {
                            mcr.keysModified.add(k);
                        }
                    }


                    mModified.clear();
                }
            }
            return mcr;
        }
```
这里需要弄清楚两个对象mMap和mModified，mMap是存放当前SharedPreferences文件中的键值对，而mModified是存放此时edit时put进去的键值对。mDiskWritesInFlight表示正在等待写的操作数量。
可以看到这个方法中首先处理了clear标志，它调用的是mMap.clear()，然后再遍历mModified将新的键值对put进mMap，也就是说在一次commit事务中，如果同时put一些键值对和调用clear后再commit，那么clear掉的只是之前的键值对，这次put进去的键值对还是会被写入的。
遍历mModified时，需要处理一个特殊情况，就是如果一个键值对的value是this（SharedPreferencesImpl）或者是null那么表示将此键值对删除，这个在remove方法中可以看到，如果之前有同样的key且value不同则用新的valu覆盖旧的value，如果没有存在同样的key则完整写入。需要注意的是这里使用了同步锁住edtor对象，保证了当前数据正确存入。
```
 public Editor remove(String key) {
            synchronized (this) {
                mModified.put(key, this);
                return this;
            }
        }


        public Editor clear() {
            synchronized (this) {
                mClear = true;
                return this;
            }
        }
```


commit接下来就是调用enqueueDiskWrite方法：
```
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };


        final boolean isFromSyncCommit = (postWriteRunnable == null);


        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }


        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }


```
定义一个Runnable任务，在Runnable中先调用writeToFile进行写操作，写操作需要先获得mWritingToDiskLock，也就是写锁。然后执行mDiskWritesInFlight–，表示正在等待写的操作减少1。
判断postWriteRunnable是否为null，调用commit时它为null，而调用apply时它不为null。isFromSyncCommit为true，而且有1个写操作需要执行，那么就调用writeToDiskRunnable.run()，注意这个调用是在当前线程中进行的。如果不是commit，那就是apply，这时调用QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable)，这个QueuedWork类其实很简单，里面有一个SingleThreadExecutor，用于异步执行这个writeToDiskRunnable，commit的写操作是在调用线程中执行的，而apply内部是用一个单线程的线程池实现的，因此写操作是在子线程中执行的。


**commit和apply的总结：**
1.  apply没有返回值而commit返回boolean表明修改是否提交成功 ；
2. commit是把内容同步提交到硬盘的，而apply先立即把修改提交到内存，然后开启一个异步的线程提交到硬盘，并且如果提交失败，你不会收到任何通知。
3. 所有commit提交是同步过程，效率会比apply异步提交的速度慢，在不关心提交结果是否成功的情况下，优先考虑apply方法。
4. apply是使用异步线程写入磁盘，commit是同步写入磁盘。所以我们在主线程使用的commit的时候，需要考虑是否会出现ANR问题。（不适合大量数据存储）




#4. 查看Sharedpreferencesd 保存数据的xml文件
 要想查看data文件首先要获取手机root权限，成功root后，修改data权限即可查看data里面的数据库。由于在xml文件内可以很清楚的查看到各个键-值”对数据，所以用Sharedpreferencesd保存比较重要的数据的时候最好先加密再保存。成功查看如下图所示：


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/3064872-9f6046f23e44141b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




![Paste_Image.png](http://upload-images.jianshu.io/upload_images/3064872-606071ffd823d554.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


data权限修改办法： 
```+
1. 打开cmd；
2. 输入’adb shell’；
3. 输入su，回车 ；
4. 输入chmod 777 /data/data/<package name>/shared_prefs 回车
该步骤设置data文件夹权限为777（drwxrwxrwx,也即administrators、power users和users组都有对该文件夹的读、写、运行权限） ;
```
**当你在Linux下用命令ll 或者ls -la的时候会看到类似drwxr-xr-x这样标识，具体代表什么意思呢?**
 这段标识总长度为10位（10个‘-’），第一位表示文件类型，如该文件是文件(用-表示），如该文件是文件夹(用d表示),如该文件是连接文件(用l表示），后面9个按照三个一组分，第一组：用户权限，第二组：组权限，第三组：其他权限。    每一组是三位，分别是读 r ,写 w,执行 x，这些权限都可以用数字来表示：r 4, w 2 , x 1。如果没有其中的某个权限则用‘-’表示。例如：
 1. -rwxrwx---，第一位‘-’代表的是文件，第二位到第四位rwx代表此文件的拥有者有读、写、执行的权限，同组用户也有读、写、及执行权限，其他用户组没任何权限。用数字来表示的话则是770.
 2. drwx------，第一位‘d’代表的是文件夹，第二位到第四位rwx代表此文件夹的拥有者有读、写、执行的权限，第五位到第七位代表的是拥有者同组用户的权限，同组用户没有任何权限，第八位到第十位代表的是其他用户的权限，其他用户也没有任何权限。用数字来表示的话则是700.
