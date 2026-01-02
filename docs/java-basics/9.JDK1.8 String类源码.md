> String类我们都不陌生，在日常的开发中我们经常使用它的一些方法，如：charAt()、trim()、replace()、compareTo()等等，可是对于它的内部实现却不知道。本文主要解析一些String实现源码。

# 定义

String的内部存储是是基于char数组的，如：

```java
public final class String
        implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
    //......
}
```

从String类的接口实现我们可以看出：

1. String类被final关键字修饰，不能被继承，创建后不可修改
2. String类实现了Serializable,可以实现序列化
3. String类实现了Comparable接口，可以比较大小
4. String类实现了CharSequence接口，本质上的存储结构为char数组

# 核心方法

## 构造方法

String构造方法有很多个，下面列举了几个比较核心的构造方法：

```java
  //1.参数为String的构造方法
public String(String original){
        this.value=original.value;
        this.hash=original.hash;
        }

//2.参数为char[]的构造方法，将字符数组转成字符串
public String(char value[]){
        this.value=Arrays.copyOf(value,value.length);
        }

//3.参数StringBuffer构造方法
public String(StringBuffer buffer){
synchronized(buffer){
        this.value=Arrays.copyOf(buffer.getValue(),buffer.length());
        }
        }

//4.参数为StringBuilder构造方法
public String(StringBuilder builder){
        this.value=Arrays.copyOf(builder.getValue(),builder.length());
        }

//5.参数为value[]，offset，count，将字符数组的一部分转成字符串
public String(char value[],int offset,int count){
        //offset是要转化的数组起始坐标，count是要转化的字符长度
        if(offset< 0){
        throw new StringIndexOutOfBoundsException(offset);
        }
        if(count<=0){
        if(count< 0){
        throw new StringIndexOutOfBoundsException(count);
        }
        if(offset<=value.length){
        this.value="".value;
        return;
        }
        }
        //如果起始位置>字符数组长度 - 个数,则无法截取到count个字符，抛异常
        if(offset>value.length-count){
        throw new StringIndexOutOfBoundsException(offset+count);
        }
        //从offset开始，截取到offset+count位置，不包括offset+count位置
        this.value=Arrays.copyOfRange(value,offset,offset+count);
        }
```

## 常用方法

**1. length()、isEmpty()**

```java
//String的长度就是value数组的长度
public int length(){
        return value.length;
        }

//判断String是否为空，为空返回true,否则返回false
public boolean isEmpty(){
        return value.length==0;
        }
```

**2. equals()**
常用的方法，用于比较两个字符串的值是否相等，内部重写了Object的equals方法,传参为Object类型，源码如下：

```java
public boolean equals(Object anObject){
        // 如果传入的对象跟当前对象引用相同，则返回 true
        if(this==anObject){
        return true;
        }
        // 判断传入的值是否为 String 类型，如果不是则直接返回 false
        if(anObject instanceof String){
        String anotherString=(String)anObject;
        int n=value.length;
        if(n==anotherString.value.length){
        // 把两个字符串都转换为 char 数组对比
        char v1[]=value;
        char v2[]=anotherString.value;
        int i=0;
        // 循环比对两个字符串的每一个字符
        while(n--!=0){
        // 如果其中有一个字符不相等就返回 false，否则继续对比
        if(v1[i]!=v2[i])
        return false;
        i++;
        }
        return true;
        }
        }
        //非string类型直接返回 false
        return false;
        }
```

此外，String类的内部还有一个equalsIgnoreCase()，它是用于忽略字符串大小写之后进行字符串比对的方法。

```java
public boolean equalsIgnoreCase(String anotherString){
        return(this==anotherString)?true
        :(anotherString!=null)
        &&(anotherString.value.length==value.length)
        &&regionMatches(true,0,anotherString,0,value.length);
        }
```

**3. CompareTo（）**
CompareTo（）方法用于比较两个字符串，它的返回值为int类型，正数为大，负数为小，基于字符的ASSIC码比较的，源码如下：

```java
public int compareTo(String anotherString){
        int len1=value.length;
        int len2=anotherString.value.length;
        // 获取到两个字符串长度最短的那个字符串的 int 值
        int lim=Math.min(len1,len2);
        char v1[]=value;
        char v2[]=anotherString.value;
        int k=0;
        // 循环对比每一个字符
        while(k<lim){
        char c1=v1[k];
        char c2=v2[k];
        if(c1!=c2){
        // 有字符不相等就返回差值
        return c1-c2;
        }
        k++;
        }
        return len1-len2;
        }
```

从上面我们可以看出，compareTo()方法和equals()方法都是用于比较两个字符串的，他们的不同点在于：

- equals()的传参是Object对象，而compareTo()是String类型
- equals()的返回值是Boolean类型，而compareTo()返回值则为int类型

除此之外，String类里还有一个，compareToIgnoreCase（）方法，它是忽略大小写的字符串比较，源码如下：

```java
public int compareToIgnoreCase(String str){
        return CASE_INSENSITIVE_ORDER.compare(this,str);
        }
```

**4. replace（），replaceAll（）**
字符串替换方法，源码如下：

```java
//将older字符全部替换为newChar
public String replace(char oldChar,char newChar){
        if(oldChar!=newChar){
        int len=value.length;
        int i=-1;
        char[]val=value; /* avoid getfield opcode */

        while(++i<len){
        if(val[i]==oldChar){
        break;
        }
        }
        if(i<len){
        //创建新数组，全部复制过去
        char buf[]=new char[len];
        for(int j=0;j<i; j++){
        buf[j]=val[j];
        }
        while(i<len){
        char c=val[i];
        buf[i]=(c==oldChar)?newChar:c;
        i++;
        }
        return new String(buf,true);
        }
        }
        return this;
        }

//替换第一个旧字符
public String replaceFirst(String regex,String replacement){
        return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
        }

//用新的字符串替换旧的字符串
public String replace(CharSequence target,CharSequence replacement){
        return Pattern.compile(target.toString(),Pattern.LITERAL).matcher(
        this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
        }

//如果没有正则表达式限制，跟replace功能一样
public String replaceAll(String regex,String replacement){
        return Pattern.compile(regex).matcher(this).replaceAll(replacement);
        }
```

**5.split()方法**

```java
//把字符串分割并返回字符串数组，其内部是通过正则来实现的
public String[]split(String regex,int limit){
        /* fastpath if the regex is a
         (1)one-char String and this character is not one of the
            RegEx's meta characters ".$|()[{^?*+\\", or
         (2)two-char String and the first char is the backslash and
            the second is not the ascii digit or ascii letter.
         */
        char ch=0;
        //1.如果regex只有一位，且不为列出的特殊字符
        if(((regex.value.length==1&&
        ".$|()[{^?*+\\".indexOf(ch=regex.charAt(0))==-1)||
        //2.如果regex的长度为2，第一位为转义字符且第二位不是数字或字母，“|”表示或，即只要ch小	 于0或者大于9任一成立，小于a或者大于z任一成立，小于A或大于Z任一成立
        (regex.length()==2&&
        regex.charAt(0)=='\\'&&
        (((ch=regex.charAt(1))-'0')|('9'-ch))< 0&&
        ((ch-'a')|('z'-ch))< 0&&
        ((ch-'A')|('Z'-ch))< 0))&&
        //3.不属于utf-16之间的字符
        (ch<Character.MIN_HIGH_SURROGATE||
        ch>Character.MAX_LOW_SURROGATE))
        {
        int off=0;
        int next=0;
        boolean limited=limit>0;
        //为了应对这几种特殊情况，使用一个ArrayList<String>存放每一段找到分割点的字符串，不断循环。
        ArrayList<String> list=new ArrayList<>();
        while((next=indexOf(ch,off))!=-1){
        if(!limited||list.size()<limit -1){
        list.add(substring(off,next));
        off=next+1;
        }else{    // last one
        //assert (list.size() == limit - 1);
        list.add(substring(off,value.length));
        off=value.length;
        break;
        }
        }
        // If no match was found, return this
        if(off==0)
        return new String[]{this};

        // Add remaining segment
        if(!limited||list.size()<limit)
        list.add(substring(off,value.length));

        // Construct result
        int resultSize=list.size();
        if(limit==0){
        while(resultSize>0&&list.get(resultSize-1).length()==0){
        resultSize--;
        }
        }
        String[]result=new String[resultSize];
        return list.subList(0,resultSize).toArray(result);
        }
        //否则，使用Pattern的正则方式去解析并拆分成字符串数组
        return Pattern.compile(regex).split(this,limit);
        }

```

**6. 其他重要方法**

```java
//查询字符串首次出现的下标位置
public int indexOf(int ch){
        return indexOf(ch,0);
        }
//字符串最后出现的下标位置
public int lastIndexOf(int ch){
        return lastIndexOf(ch,value.length-1);
        }
//字符串是否包含另一个字符串
public boolean contains(CharSequence s){
        return indexOf(s.toString())>-1;
        }
//将字符串转换为大小写
public String toLowerCase(){
        return toLowerCase(Locale.getDefault());
        }
public String toUpperCase(){
        return toUpperCase(Locale.getDefault());
        }
//String首尾空格
public String trim(){
        int len=value.length;
        int st=0;
        char[]val=value;    /* avoid getfield opcode */

        while((st<len)&&(val[st]<=' ')){
        st++;
        }
        while((st<len)&&(val[len-1]<=' ')){
        len--;
        }
        return((st>0)||(len<value.length))?substring(st,len):this;
        }
//检测是否匹配给定的正则表达式
public boolean matches(String regex){
        return Pattern.matches(regex,this);
        }
//String转Char数组并返回
public char[]toCharArray(){
        // Cannot use Arrays.copyOf because of class initialization order issues
        //String 和Arrays 都属于rt.jar中的类，但是BootstrapClassloader 在加载这两个类的顺序是不同的。
        //所以当String.class被加载进内存的时候,Arrays此时没有被加载，所以直接使用肯定会抛异常。而System.arrayCopy是使用native代码，则不会有这个问题。
        char result[]=new char[value.length];
        System.arraycopy(value,0,result,0,value.length);
        return result;
        }
//把字符串数组转化为字符串
public static String join(CharSequence delimiter,CharSequence...elements){
        Objects.requireNonNull(delimiter);
        Objects.requireNonNull(elements);
        // Number of elements not likely worth Arrays.stream overhead.
        StringJoiner joiner=new StringJoiner(delimiter);
        for(CharSequence cs:elements){
        joiner.add(cs);
        }
        return joiner.toString();
        }
```

# 知识扩展

**1. ==和equals的区别**
对于基本数据来说，用于比较两个“值”是否相等，而对于引用类型来说，用于比较引用地址是否相同。Object中的equals方法就是==，而String重写了equals()方法，把它修改为比较两个字符串的值是否相等。如上equals方法。
**2.使用final修饰的好处**
从上面开头写的，我们可以看见String是被final修饰的不可继承类，这样有两个好处:

- 安全 当你在调用其他方法时，比如调用一些系统级操作指令之前，可能会有一些列校验，如果是可变类的话，当年校验完之后，内部值又被修改了，可能会引起系统级崩溃问题
- 高效 传参的时候不需要考虑它会被哪个修改，如果是可变类，可能需要重新拷贝出来一个新值进行传参，这会在性能上造成一定的损耗

**3.String、StringBuilder、StringBuffer的区别**
String是不可变的类，因此在做字符串拼接的时候性能比较差，字符串拼接一般通过StringBuilder或者StringBuffer来实现，他们提供了append和insert方法可用于字符串的拼接。StringBuilder是线程不安全的，单线程下比较快，而StringBuffer通过JVM的synchronized来保证线程安全，性能不是很高，源码如下：

```java
@Override
public synchronized StringBuffer append(Object obj){
        toStringCache=null;
        super.append(String.valueOf(obj));
        return this;
        }
```

# 总结

本文主要介绍了String类里面常用的几个方法的源码，也扩展了关于String的一些思考问题。有许多类似的方法没有一一列举出来，感兴趣的朋友可以打开String类深挖一下。