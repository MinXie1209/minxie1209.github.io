## IO

输入流 输出流

### 字节流

字节流都是继承InputStream\OutputStream,如FileInputStream

```java
class Test1 {

    @Test
    void test1() throws FileNotFoundException {
        URL resourceIn = ClassLoader.getSystemResource("testIn.txt");
        URL resourceOut = ClassLoader.getSystemResource("testOut.txt");

        try (InputStream in = new FileInputStream(resourceIn.getPath());
             OutputStream out = new FileOutputStream(resourceOut.getPath());) {
            int read = -1;
            while ((read = in.read()) != -1) {
                out.write(read);
            }
            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

### 字符流

字符流继承Reader\Writer, BufferedReader.readLine()可以一行一行读取字符

```java
class Test1 {

    @Test
    void test2() throws FileNotFoundException {
        URL resourceIn = ClassLoader.getSystemResource("testIn.txt");
        URL resourceOut = ClassLoader.getSystemResource("testOut.txt");

        try (Reader in = new FileReader(resourceIn.getPath());
             Writer out = new FileWriter(resourceOut.getPath());) {
            int read = -1;
            while ((read = in.read()) != -1) {
                out.write(read);
            }
            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    @Test
    void test3() throws FileNotFoundException {
        URL resourceIn = ClassLoader.getSystemResource("testIn.txt");
        URL resourceOut = ClassLoader.getSystemResource("testOut.txt");

        try (Reader reader = new FileReader(resourceIn.getPath());
             Writer writer = new FileWriter(resourceOut.getPath());
             BufferedReader in = new BufferedReader(reader);
             BufferedWriter out = new BufferedWriter(writer)) {
            String read = null;
            while ((read = in.readLine()) != null) {
                out.write(read);
                out.newLine();
            }
            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

> 获取resources下文件的方式

```java
class Test1 {

    @Test
    void getResources() {
        ClassLoader c1 = Thread.currentThread().getContextClassLoader();
        URL resource1 = c1.getResource("test1.txt");
        System.out.println(resource1);

        ClassLoader c2 = Demo2244AppsslicationTests.class.getClassLoader();
        URL resource2 = c2.getResource("test1.txt");
        System.out.println(resource2);

        ClassLoader c3 = ClassLoader.getSystemClassLoader();
        URL resource3 = c3.getResource("test1.txt");
        System.out.println(resource3);

        URL resource4 = ClassLoader.getSystemResource("test1.txt");
        System.out.println(resource4);
    }

}

```
