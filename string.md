## 字符串处理



以实现 mybatis-plus 多字段排序为需求，把驼峰命名转下划线命名

 方式一：

``` java
/**
 * 把驼峰命名转下划线
 *
 * @param str
 * @return
 */
public static String convertToLowerCase(String str) {
    StringBuilder sb = new StringBuilder(str);
    char[] charsArray = str.toCharArray();
    int num = 0;
    for (int i = 0; i < charsArray.length; i++) {
        if (i != 0 && Character.isUpperCase(charsArray[i])) {
            sb.insert(i + num, "_");
            num++;
        }
    }
    return sb.toString().toLowerCase();
}
```

方式二：

``` java
/**
 * 把驼峰命名转下划线
 *
 * @param str
 * @return
 */
public static String convertToLowerCase(String str) {
    String test = str.replaceAll("([a-z])([A-Z])", "$1_$2");
    return test.toLowerCase();
}
```



