---
title: java String
date: 2018/3/16 08:28:25
category:
- java基础
- java基础
tag:

comments: true  
---
## String基础 ##
### 不可变String ###
String对象是不可变的,每一个修改String值的方法,实际上都是创建了一个新的String对象,原初的String不变

### API ###

     String() 
     初始化一个新创建的 String 对象，使其表示一个空字符序列。
     String(byte[] bytes) 
     通过使用平台的默认字符集解码指定的 byte 数组，构造一个新的 String。
     String(byte[] bytes, Charset charset) 
     通过使用指定的 charset 解码指定的 byte 数组，构造一个新的 String。
     String(byte[] ascii, int hibyte) 
     已过时。 该方法无法将字节正确地转换为字符。从 JDK 1.1 开始，完成该转换的首选方法是使用带有 Charset、字符集名称，或使用平台默认字符集的 String 构造方法。
     String(byte[] bytes, int offset, int length) 
     通过使用平台的默认字符集解码指定的 byte 子数组，构造一个新的 String。
     String(byte[] bytes, int offset, int length, Charset charset) 
     通过使用指定的 charset 解码指定的 byte 子数组，构造一个新的 String。
     String(byte[] ascii, int hibyte, int offset, int count) 
     已过时。 该方法无法将字节正确地转换为字符。从 JDK 1.1 开始，完成该转换的首选方法是使用带有 Charset、字符集名称，或使用平台默认字符集的 String 构造方法。
     String(byte[] bytes, int offset, int length, String charsetName) 
     通过使用指定的字符集解码指定的 byte 子数组，构造一个新的 String。
     String(byte[] bytes, String charsetName) 
     通过使用指定的 charset 解码指定的 byte 数组，构造一个新的 String。
     String(char[] value) 
     分配一个新的 String，使其表示字符数组参数中当前包含的字符序列。
     String(char[] value, int offset, int count) 
     分配一个新的 String，它包含取自字符数组参数一个子数组的字符。
     String(int[] codePoints, int offset, int count) 
     分配一个新的 String，它包含 Unicode 代码点数组参数一个子数组的字符。
     String(String original) 
     初始化一个新创建的 String 对象，使其表示一个与参数相同的字符序列；换句话说，新创建的字符串是该参数字符串的副本。
     String(StringBuffer buffer) 
     分配一个新的字符串，它包含字符串缓冲区参数中当前包含的字符序列。
     String(StringBuilder builder) 
     分配一个新的字符串，它包含字符串生成器参数中当前包含的字符序列。

     方法摘要
     char	charAt(int index) 
     返回指定索引处的 char 值。
     int	codePointAt(int index) 
     返回指定索引处的字符（Unicode 代码点）。
     int	codePointBefore(int index) 
     返回指定索引之前的字符（Unicode 代码点）。
     int	codePointCount(int beginIndex, int endIndex) 
     返回此 String 的指定文本范围中的 Unicode 代码点数。
     int	compareTo(String anotherString) 
     按字典顺序比较两个字符串。
     int	compareToIgnoreCase(String str) 
     按字典顺序比较两个字符串，不考虑大小写。
     String	concat(String str) 
     将指定字符串连接到此字符串的结尾。
     boolean	contains(CharSequence s) 
     当且仅当此字符串包含指定的 char 值序列时，返回 true。
     boolean	contentEquals(CharSequence cs) 
     将此字符串与指定的 CharSequence 比较。
     boolean	contentEquals(StringBuffer sb) 
     将此字符串与指定的 StringBuffer 比较。
     static String	copyValueOf(char[] data) 
     返回指定数组中表示该字符序列的 String。
     static String	copyValueOf(char[] data, int offset, int count) 
     返回指定数组中表示该字符序列的 String。
     boolean	endsWith(String suffix) 
     测试此字符串是否以指定的后缀结束。
     boolean	equals(Object anObject) 
     将此字符串与指定的对象比较。
     boolean	equalsIgnoreCase(String anotherString) 
     将此 String 与另一个 String 比较，不考虑大小写。
     static String	format(Locale l, String format, Object... args) 
     使用指定的语言环境、格式字符串和参数返回一个格式化字符串。
     static String	format(String format, Object... args) 
     使用指定的格式字符串和参数返回一个格式化字符串。
     byte[]	getBytes() 
     使用平台的默认字符集将此 String 编码为 byte 序列，并将结果存储到一个新的 byte 数组中。
     byte[]	getBytes(Charset charset) 
     使用给定的 charset 将此 String 编码到 byte 序列，并将结果存储到新的 byte 数组。
     void	getBytes(int srcBegin, int srcEnd, byte[] dst, int dstBegin) 
     已过时。 该方法无法将字符正确转换为字节。从 JDK 1.1 起，完成该转换的首选方法是通过 getBytes() 方法，该方法使用平台的默认字符集。
     byte[]	getBytes(String charsetName) 
     使用指定的字符集将此 String 编码为 byte 序列，并将结果存储到一个新的 byte 数组中。
     void	getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin) 
     将字符从此字符串复制到目标字符数组。
     int	hashCode() 
     返回此字符串的哈希码。
     int	indexOf(int ch) 
     返回指定字符在此字符串中第一次出现处的索引。
     int	indexOf(int ch, int fromIndex) 
     返回在此字符串中第一次出现指定字符处的索引，从指定的索引开始搜索。
     int	indexOf(String str) 
     返回指定子字符串在此字符串中第一次出现处的索引。
     int	indexOf(String str, int fromIndex) 
     返回指定子字符串在此字符串中第一次出现处的索引，从指定的索引开始。
     String	intern() 
     返回字符串对象的规范化表示形式。
     boolean	isEmpty() 
     当且仅当 length() 为 0 时返回 true。
     int	lastIndexOf(int ch) 
     返回指定字符在此字符串中最后一次出现处的索引。
     int	lastIndexOf(int ch, int fromIndex) 
     返回指定字符在此字符串中最后一次出现处的索引，从指定的索引处开始进行反向搜索。
     int	lastIndexOf(String str) 
     返回指定子字符串在此字符串中最右边出现处的索引。
     int	lastIndexOf(String str, int fromIndex) 
     返回指定子字符串在此字符串中最后一次出现处的索引，从指定的索引开始反向搜索。
     int	length() 
     返回此字符串的长度。
     boolean	matches(String regex) 
     告知此字符串是否匹配给定的正则表达式。
     int	offsetByCodePoints(int index, int codePointOffset) 
     返回此 String 中从给定的 index 处偏移 codePointOffset 个代码点的索引。
     boolean	regionMatches(boolean ignoreCase, int toffset, String other, int ooffset, int len) 
     测试两个字符串区域是否相等。
     boolean	regionMatches(int toffset, String other, int ooffset, int len) 
     测试两个字符串区域是否相等。
     String	replace(char oldChar, char newChar) 
     返回一个新的字符串，它是通过用 newChar 替换此字符串中出现的所有 oldChar 得到的。
     String	replace(CharSequence target, CharSequence replacement) 
     使用指定的字面值替换序列替换此字符串所有匹配字面值目标序列的子字符串。
     String	replaceAll(String regex, String replacement) 
     使用给定的 replacement 替换此字符串所有匹配给定的正则表达式的子字符串。
     String	replaceFirst(String regex, String replacement) 
     使用给定的 replacement 替换此字符串匹配给定的正则表达式的第一个子字符串。
     String[]	split(String regex) 
     根据给定正则表达式的匹配拆分此字符串。
     String[]	split(String regex, int limit) 
     根据匹配给定的正则表达式来拆分此字符串。
     boolean	startsWith(String prefix) 
     测试此字符串是否以指定的前缀开始。
     boolean	startsWith(String prefix, int toffset) 
     测试此字符串从指定索引开始的子字符串是否以指定前缀开始。
     CharSequence	subSequence(int beginIndex, int endIndex) 
     返回一个新的字符序列，它是此序列的一个子序列。
     String	substring(int beginIndex) 
     返回一个新的字符串，它是此字符串的一个子字符串。
     String	substring(int beginIndex, int endIndex) 
     返回一个新字符串，它是此字符串的一个子字符串。
     char[]	toCharArray() 
     将此字符串转换为一个新的字符数组。
     String	toLowerCase() 
     使用默认语言环境的规则将此 String 中的所有字符都转换为小写。
     String	toLowerCase(Locale locale) 
     使用给定 Locale 的规则将此 String 中的所有字符都转换为小写。
     String	toString() 
     返回此对象本身（它已经是一个字符串！）。
     String	toUpperCase() 
     使用默认语言环境的规则将此 String 中的所有字符都转换为大写。
     String	toUpperCase(Locale locale) 
     使用给定 Locale 的规则将此 String 中的所有字符都转换为大写。
     String	trim() 
     返回字符串的副本，忽略前导空白和尾部空白。
     static String	valueOf(boolean b) 
     返回 boolean 参数的字符串表示形式。
     static String	valueOf(char c) 
     返回 char 参数的字符串表示形式。
     static String	valueOf(char[] data) 
     返回 char 数组参数的字符串表示形式。
     static String	valueOf(char[] data, int offset, int count) 
     返回 char 数组参数的特定子数组的字符串表示形式。
     static String	valueOf(double d) 
     返回 double 参数的字符串表示形式。
     static String	valueOf(float f) 
     返回 float 参数的字符串表示形式。
     static String	valueOf(int i) 
     返回 int 参数的字符串表示形式。
     static String	valueOf(long l) 
     返回 long 参数的字符串表示形式。
     static String	valueOf(Object obj) 
     返回 Object 参数的字符串表示形式。

### 格式化输出 ###
类似c/c++

	System.out.printf("Row 1: [%d %f]",x,y);
	System.out.format("Row 1: [%d %f]",x,y);
### Formatter ###
	%[argument_index$][flags][width][.precision]conversion 
	%[参数索引][标识集][输出宽度][.限制字符]标明如何格式化字符
	format(String format, Object... args)
	// 使用指定格式字符串和参数将一个格式化字符串写入此对象的目标文件中

	Formatter f=new Formatter(System.out);
	f.format("%15s %5s %10.2f",name, qut, price );

> 转换  参数类别  说明  
b  Boolean值
h  调用 Integer.toHexString(arg.hashCode()) 得到的结果。  
s  调用 arg.toString() 得到的结果。  
c  字符  结果是一个 Unicode 字符  
d  十进制整数  
o  八进制整数  
x  十六进制整数  
e  结果被格式化为用计算机科学记数法表示的十进制数  
f  结果被格式化为十进制数  
g  根据精度和舍入运算后的值，使用计算机科学记数形式或十进制格式对结果进行格式化。  
%  字面值 '%' 
 
## 正则表达式 ##
### split() ###
将字符串按从正则表达式地方切开
	
	 String[]   split(String regex) 
	 根据给定正则表达式的匹配拆分此字符串。
	 String[]   split(String regex, int limit) 
	 根据匹配给定的正则表达式来拆分此字符串。
	 String replace(char oldChar, char newChar) 
	 返回一个新的字符串，它是通过用 newChar 替换此字符串中出现的所有 oldChar 得到的。
	 String replace(CharSequence target, CharSequence replacement) 
	 使用指定的字面值替换序列替换此字符串所有匹配字面值目标序列的子字符串。
	 String replaceAll(String regex, String replacement) 
	 使用给定的 replacement 替换此字符串所有匹配给定的正则表达式的子字符串。
	 String replaceFirst(String regex, String replacement) 
	 使用给定的 replacement 替换此字符串匹配给定的正则表达式的第一个子字符串。

### regex包 ###
在regex包中，包括了两个类，Pattern(模式类)和Matcher(匹配器类)。Pattern类是用来表达和陈述所要搜索模式的对象，Matcher类是真正影响搜索的对象。另加一个新的例外类，PatternSyntaxException，当遇到不合法的搜索模式时，会抛出例外。

用法:

	Pattern p=Pattern.compile(String regex);
	Matcher m=p.matcher(String args);
	
![](http://i.imgur.com/Hx6VMIC.png)
![](http://i.imgur.com/hEXQ5vG.png)
![](http://i.imgur.com/oX0HcoK.png)
![](http://i.imgur.com/HNX8FI1.png)


### 基础知识 ###
	字符 
	x 字符 x 
	\\ 反斜线字符 
	\0n 带有八进制值 0 的字符 n (0 <= n <= 7) 
	\xhh 带有十六进制值 0x 的字符 hh 
	\t 制表符 ('\u0009') 
	\n 新行（换行）符 ('\u000A') 
	  
	字符类 
	[abc] a、b 或 c（简单类） 
	[^abc] 任何字符，除了 a、b 或 c（否定） 
	[a-zA-Z] a 到 z 或 A 到 Z，两头的字母包括在内（范围） 
	[a-d[m-p]] a 到 d 或 m 到 p：[a-dm-p]（并集） 
	[a-z&&[def]] d、e 或 f（交集） 
	[a-z&&[^bc]] a 到 z，除了 b 和 c：[ad-z]（减去） 
	[a-z&&[^m-p]] a 到 z，而非 m 到 p：[a-lq-z]（减去） 
	  
	预定义字符类 
	. 任何字符
	\d 数字：[0-9] 
	\D 非数字： [^0-9] 
	\s 空白字符：[ \t\n\x0B\f\r] 
	\S 非空白字符：[^\s] 
	\w 单词字符：[a-zA-Z_0-9] 
	\W 非单词字符：[^\w] 
	  
	边界匹配器 
	^ 行的开头 
	$ 行的结尾 
	  
	数量词 
	X? 一次或一次也没有 
	X* 零次或多次 
	X+ 一次或多次 
	X{n} 恰好 n 次 
	X{n,} 至少 n 次 
	X{n,m} 至少 n 次，但是不超过 m 次 
	  
	Logical 运算符 
	XY X 后跟 Y 
	X|Y X 或 Y 
	(X) X，作为捕获组 
	  
### 举例 ###
	import java.util.regex.Matcher;
	import java.util.regex.Pattern;
	
	public class Test {
	
	    public static void main(String[] args) {
	        p("abc".matches("..."));
	        p("a2389a".replaceAll("\\d", "*"));
	        Pattern p = Pattern.compile("[a-z]{3}");
	        Matcher m = p.matcher("abc");
	        p(m.matches());
	        //上面的三行代码可以用下面一行代码代替
	        p("abc".matches("[a-z]{3}"));
	    }
	    
	    public static void p(Object o){
	        System.out.println(o);
	    }
	}
	
	/**output
	* true
	* a****a
	* true
	* true
	**/

