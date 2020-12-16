---
title: FileProvider
date: 2019-08-18
categories: Android
---



从 Android N（7.0） 开始，将严格执行 StrictMode 模式，也就是说，将对安全做更严格的校验。而从 Android N 开始，将不允许在 App 间，使用 `file://` 的方式，传递一个 File ，否者会抛出 FileUriExposedException 的错误，会直接引发 Crash。只能使用FileProvider 将` file://` 替换为 `content://`。



FileProvider 本质上就是一个 ContentProvider ，它其实也继承了 ContentProvider 的特性。ContentProvider 其实就是在可控的范围内，向外部其他的 App 分享数据。而 FileProvider 将这样的数据变成了一个 File 文件而已。



下面的代码演示了如何使用 FileProvier：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <application
        ...>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.myapp.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
        ...
    </application>
</manifest>
```



provider 标签下，配置了几个属性：

- name ：配置当前 FileProvider 的实现类。
- authorities：配置一个 FileProvider 的名字，**它在当前系统内需要是唯一值**。
- exported：表示该 FileProvider 是否需要公开出去，这里不需要，所以是 false。
- granUriPermissions：是否允许授权文件的临时访问权限。这里需要，所以是 true。



在配置 Provider 的时候，还需要额外配置一个 `<meta-data/>` 标签，它用于配置 FileProvider 支持分享出去的目录。这个 `<meta-data/>` 标签的 **name  值是固定的**，resource 需要指向一个 根节点为 `paths` 的 xml 资源文件。在 `src/main/res/xml/`文件夹下面:

```xml
<paths>
    <!-- 表示 files/images/myimages 下的路径 -->
    <files-path path="images/" name="myimages" />
</paths>
```

path下面可以配置多个节点：

- **root-path**：表示根目录，『/』。

- **files-path**：表示 content.getFileDir() 获取到的目录。

- **cache-path**：表示 content.getCacheDir() 获取到的目录

- **external-path**：表示Environment.getExternalStorageDirectory() 指向的目录。

- **external-files-path**：表示 ContextCompat.getExternalFilesDirs() 获取到的目录。

- **external-cache-path**：表示 ContextCompat.getExternalCacheDirs() 获取到的目录。

具体可以参考 `FileProvider`  的官方说明文档 。



定义好需要分享的文件之后，在ClientApp请求文件的时候，我们就可以返回一个 Uri 给它。

```kotlin
val requestFile = File(imageFilenames[position])
val fileUri: Uri? = try {
    FileProvider.getUriForFile(
        this@MainActivity,
        "com.example.myapp.fileprovider",
        requestFile)
} catch (e: IllegalArgumentException) {
    Log.e("File Selector",
          "The selected file can't be shared: $requestFile")
        null
}
```

获取文件的 Uri 一定要使用 `FileProvider.getUriForFile` 这个方法，**不要使用`Uri.fromFile()`这个方法**。因为这个方法要求 ClientApp 具有读权限，并且无法跨App分享文件。



在返回Uri之后，还需要给 Intent 加上临时权限：

```kotlin
// Grant temporary read permission to the content URI
resultIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
// Put the Uri and MIME type in the result Intent
resultIntent.setDataAndType(fileUri, contentResolver.getType(fileUri))
// Set the result
setResult(Activity.RESULT_OK, resultIntent)
```



还有另外一种授权方式，但是不建议使用：

使用 `Context.grantUriPermission()` 为其他 App 授予 Uri 对象的访问权限。这种情况下，授权的有效期限，从授权一刻开始，**截止于设备重启或者手动调用 `Context.revokeUriPermission()` 方法，才会收回对此 Uri 的授权。**

而使用 Flag 的方式，当 ClientApp 的任务栈结束的时候，就会自动收回权限。



ClientApp 获取到 Uri 之后，就可以访问文件了：

```kotlin
    // Get the file's content URI from the incoming Intent
    returnIntent.data?.also { returnUri ->
        /*
         * Try to open the file for "read" access using the
         * returned URI. If the file isn't found, write to the
         * error log and return.
         */
        inputPFD = try {
            /*
             * Get the content resolver instance for this context, and use it
             * to get a ParcelFileDescriptor for the file.
             */
            contentResolver.openFileDescriptor(returnUri, "r")
        } catch (e: FileNotFoundException) {
            e.printStackTrace()
            Log.e("MainActivity", "File not found.")
            return
        }

        // Get a regular file descriptor for the file
        val fd = inputPFD.fileDescriptor
        ...
    }
```

