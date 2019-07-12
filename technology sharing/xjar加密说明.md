#     Spring Boot JAR 安全加密运行工具：xjar
文档作者：史成成，使用过程出现问题可与我联系</br>
[toc]

---
##  1. xjar功能介绍
 基于对JAR包内资源的加密以及拓展ClassLoader来构建的一套程序加密的方案，避免源码泄露或反编译。
## 2. xjar使用方法
(1):新建maven工程，添加以下依赖

	<project>
	    <!-- 设置 jitpack.io 仓库 -->
	    <repositories>
	        <repository>
	            <id>jitpack.io</id>
	            <url>https://jitpack.io</url>
	        </repository>
	    </repositories>
	    <!-- 添加 XJar 依赖 -->
	    <dependencies>
	        <dependency>
	            <groupId>com.github.core-lib</groupId>
	            <artifactId>xjar</artifactId>
	            <version>v2.0.5</version>
	        </dependency>
	    </dependencies>
	</project>
(2)编写main方法

    public static void main(String[] args) throws Exception {
        XBoot.encrypt(
                "D:\\jar\\read\\fbs-0.0.1-SNAPSHOT.jar",
                "D:\\jar\\save\\fbs-0.0.1-SNAPSHOT.jar",
                "taoqicar",
                "DES",
                56,
                56,
                new XEntryFilter<JarArchiveEntry>() {
                    public boolean filtrate(JarArchiveEntry entry) {
                        String name = entry.getName();
                        String pkg = "com/taoqicar/fbs";
                        return name.startsWith(pkg);
                    }
                }
        );

        System.out.println("end");
    }

##  3. xjar加密原理
利用Apache Commons Compress进行jar的读写操作，利用jdk内置的几种算法进行加密。</br>
下边是加密的核心方法介绍：

	public void encrypt(XKey key, InputStream in, OutputStream out) throws IOException {
       		...
            while ((entry = zis.getNextJarEntry()) != null) {
                
                // DIR ENTRY，文件夹无需加密，直接放入zos即可
                if (entry.isDirectory()) {...}
                   
                // META-INF/MANIFEST.MF  修改jar的描述文件，主要是更改类加载器，修改后写入zos
                else if (entry.getName().equals(META_INF_MANIFEST)) {...}
 
                // BOOT-INF/classes/**  对SpringBoot类型的jar包里BOOT-INF/classes/下的class文件进行加密，可以通过过滤器选择性的加密
                else if (entry.getName().startsWith(BOOT_INF_CLASSES)) {}

                // BOOT-INF/lib/**   对SpringBoot类型的jar包里BOOT-INF/lib/下引用的普通第三方jar进行加密
                else if (entry.getName().startsWith(BOOT_INF_LIB)) {...}

                // OTHER  原样写出org.springframework.boot.loader包下的所有class文件,静态文件
                else {...}
                    
                zos.closeArchiveEntry();
            }

            // 为累类加载器解密提供参照，加密的类名均作为字符串被写入到了BOOT-INF/classes/XJAR-INF/INDEXES.IDX文件内
            if (!indexes.isEmpty()) {...}
 			
			...
            // 往JAR包中注入XJar框架的classes
            if (mainClass != null) {
                XInjector.inject(zos);
            }
		...
	
    }

##  3. xjar类加载器解密
xjar通过更换jar包的Main-Class为io.xjar.boot.XJarLauncher来使用xjar本身重写的类加载器完成解密。</br>
(1)XJarLauncher的main方法如下：

    public static void main(String[] args) throws Exception {
        new XJarLauncher(args).launch();
    }
在实例化XJarLauncher时会创建一个io.xjar.XLauncher对象，XLauncher获得解密参数。</br>
Spring-Boot Jar 启动器 : XJarLauncher

    public void launch() throws Exception {
        launch(xLauncher.args);
    }

    @Override
    protected ClassLoader createClassLoader(URL[] urls) throws Exception {
        return new XBootClassLoader(urls, this.getClass().getClassLoader(), xLauncher.xDecryptor, xLauncher.xEncryptor, xLauncher.xKey);
    }
(2)自定义类加载器 : XBootClassLoader

    @Override
    public URL findResource(String name) {
        URL url = super.findResource(name);
        if (url == null) {
            return null;
        }
        try {
            return new URL(url.getProtocol(), url.getHost(), url.getPort(), url.getFile(), xBootURLHandler);
        } catch (MalformedURLException e) {
            return url;
        }
    }

	@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            return super.findClass(name);
        } catch (ClassFormatError e) {
            URL resource = findResource(name.replace('.', '/') + ".class");
            if (resource == null) {
                throw new ClassNotFoundException(name, e);
            }
            try (InputStream in = resource.openStream()) {
                ByteArrayOutputStream bos = new ByteArrayOutputStream();
                XKit.transfer(in, bos);
                byte[] bytes = bos.toByteArray();
                return defineClass(name, bytes, 0, bytes.length);
            } catch (Throwable t) {
                throw new ClassNotFoundException(name, t);
            }
        }
    }

(4)加密的URL处理器 : XBootURLHandler

    @Override
    protected URLConnection openConnection(URL url) throws IOException {
        URLConnection urlConnection = super.openConnection(url);
        return indexes.contains(url.toString())
                && urlConnection instanceof JarURLConnection
                ? new XBootURLConnection((JarURLConnection) urlConnection, xDecryptor, xEncryptor, xKey)
                : urlConnection;
    }

(5)加密的URL连接 : XBootURLConnection

    @Override
    public InputStream getInputStream() throws IOException {
        InputStream in = jarURLConnection.getInputStream();
        return xDecryptor.decrypt(xKey, in);
    }

### 参考链接
[xjar的GitHub地址](https://github.com/core-lib/xjar)</br>
[Apache Commons Compress](http://commons.apache.org/proper/commons-compress/examples.html)</br>
[spring boot应用启动原理分析](https://blog.csdn.net/hengyunabc/article/details/50120001)</br>



