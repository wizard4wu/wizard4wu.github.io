---
title: Bit位用于Option的总结
key: AAA-2021-11-16-bit-option
tags: [bit, option, java]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
lightbox: true
---

## 1.应用初衷

在日常的开发中，会产生各种各样的开关应用，开关对于某些功能的应用。如果对于每一个开关都要搞一个字段存储在字段中，这样显得特别笨重，而且占用存储内存。此时Bit位便应用而生。

<font color=red>Note:</font>建议大家int类型的存储32 - 1个开关，因为第32位是符号位，可能在有些情况下会导致一些不必要的麻烦，另外存储option的数据库字段数据库的默认值不可以设置成负数。

## 2.基本应用
使用bit位存储option只会存在两点：1.十进制数转成bit为体现option(1/0); 2.option体现bit位转成十进制数。

```java
@Data
public class Option {

    private Boolean muteUser;

    private boolean disableUser;

    public int toValue(){
        int value = 0;
        if(disableUser){
          value = set(value, 0);
        }
        if( null != muteUser && muteUser){
            value = set(value, 1);
        }
        return value;
    }

    public Option buildOption( int value){
        this.disableUser = isSet(value, 0);
        this.muteUser = isSet(value, 1);
        return this;
    }

    public static int set(int option, int bitIndex) {
        int value = 1 << bitIndex;
        return option | value;
    }

    public static int unset(int option, int bitIndex) {
        int value = 1 << bitIndex;
        return option & (~ value);
    }

    public static boolean isSet(int option, int bitIndex){
        int value = 1 << bitIndex;
        return value == (option & value);
    }

    //实际业务应用, 以需要更新的为基础并与来自数据库的结合
    public Option combineOption(Option optionFromDB, Option needSetOption){
        boolean muteUser = optionFromDB.getMuteUser();

        boolean disableUser = optionFromDB.isDisableUser();

        if(disableUser){
            needSetOption.setDisableUser(disableUser);
        }
        if(null == needSetOption.getMuteUser()){
            needSetOption.setMuteUser(muteUser);
        }
        return needSetOption;
    }

}
```

上述代码可以完成option中两个开关的设置，但是可以发现两个开关类型是不一样的。大布尔可以实现开关的可逆性，就是能开也能关；小布尔只能实现开关的单向性，只能开启不能关闭。

<font color=red>原因分析：</font>

由于在设置某个option时，我们必须先获取历史option(可以理解成DB的数据)，再将需要设置的option和历史的option组合在一起形成一个value存储起来。小布尔的默认值是false，这样就无法分辨这个option中的小布尔值是用户设置的还是其默认的；而大布尔默认是null，如果用户设置为false，大布尔可以显示false，这样就可以完成option的开闭。

## 3.进阶应用

受Effective java中的第36条(使用EnumSet来替换Bit域)和公司同事的代码启发，使用枚举set来存储各个option的枚举进而方便实现option的开闭。

```java
@Data
public class OptionPro {
    private EnumSet<OptionProEnum> optionProEnumSet;

    public boolean disableUser(){
        return null != optionProEnumSet && optionProEnumSet.contains(OptionProEnum.DISABLE_USER);
    }

    public boolean muteUser(){
        return null != optionProEnumSet && optionProEnumSet.contains(OptionProEnum.MUTE_USER);
    }

    public void removeOption(Set<OptionProEnum> enumSet){
        this.getOptionProEnumSet().removeAll(enumSet);
    }

    public void addOption(Set<OptionProEnum> enumSet){
        this.getOptionProEnumSet().addAll(enumSet);
    }

    public static boolean isSet(int option, int bitIndex){
        int value = 1 << bitIndex;
        return value == (option & value);
    }

    public int toValue(){
        int value = 0;
        Set<OptionProEnum> optionProEnumSet = this.getOptionProEnumSet();
        Iterator<OptionProEnum> optionProEnumIterator = optionProEnumSet.iterator();
        while (optionProEnumIterator.hasNext()){
            int bitIndex = optionProEnumIterator.next().getBitIndex();
            if( bitIndex >= 0 && bitIndex < Integer.SIZE - 1){
                value |= 1 << bitIndex;
            }
        }
        return value;
    }

    public Set<OptionProEnum> buildEnumSet(int value){
        Set<OptionProEnum> set = EnumSet.noneOf(OptionProEnum.class);
        OptionProEnum[] enums = OptionProEnum.class.getEnumConstants();
       return Arrays.stream(enums).filter(e -> isSet(value, e.getBitIndex())).collect(Collectors.toSet());
    }

    public Set<OptionProEnum> combineOptionPro(Set<OptionProEnum> optionFromDB, Set<OptionProEnum> addOption, Set<OptionProEnum> removeOption){
        optionFromDB.addAll(addOption);
        optionFromDB.removeAll(removeOption);
        return optionFromDB;
    }

    public enum OptionProEnum {
        DISABLE_USER(0), MUTE_USER(1);

        private int bitIndex;

        OptionProEnum(int bitIndex) {
            this.bitIndex = bitIndex;
        }

        public int getBitIndex() {
            return bitIndex;
        }
    }
}

```

<font color = red>Note:</font>上述一些方法可以使用泛型方式抽成共用的方法用于整个项目中的应用。
## 4.项目实战

由于在项目中使用时，会存在缓存的读取和写入操作，那么这时序列化和反序列化就是重点了，如何让整个过程性能消耗最低呢？我给出了以下的解决方案。

```java

//定义一个Util方法 用于int <-----> enumset。
public class EnumUtil<E> {

    public static  <E extends Enum<E> & EnumIndex> int toValue(EnumSet<E> enumSet){
        int value = 0;
        Iterator<E> optionProEnumIterator = enumSet.iterator();
        while (optionProEnumIterator.hasNext()){
            int bitIndex = optionProEnumIterator.next().getBitIndex();
            if( bitIndex >= 0 && bitIndex < Integer.SIZE - 1){
                value |= 1 << bitIndex;
            }
        }
        return value;
    }

    public static <E extends Enum<E> & EnumIndex> EnumSet<E> buildEnumSet(int value, Class<E> enumClass){
        EnumSet<E> enumSet = EnumSet.noneOf(enumClass);
        E[] enums = enumClass.getEnumConstants();
        Arrays.stream(enums).filter(e -> isSet(value, e.getBitIndex())).forEach(enumValue -> enumSet.add(enumValue));
        return enumSet;
    }

    public static boolean isSet(int option, int bitIndex){
        int value = 1 << bitIndex;
        return value == (option & value);
    }
}

//定义接口，用于在上述的util方法中获取bit位
public interface EnumIndex<E> {
    int getBitIndex();
}


@Data
public class OptionProject {

    private Integer value;

    @JsonIgnore
    private EnumSet<OptionProject.OptionProjectEnum> optionProEnumSet;

    public static void main(String[] args) throws JsonProcessingException {
        OptionProject project = new OptionProject();
        project.setValue(6);

        ObjectMapper objectMapper = new ObjectMapper();
        String jsonString = objectMapper.writeValueAsString(project);
        System.out.println(jsonString);

        OptionProject optionProject = objectMapper.readValue(jsonString, OptionProject.class);
        System.out.println(optionProject.activeUser());
    }

    private EnumSet<OptionProject.OptionProjectEnum> retriveOptionProjectEnumSet() {
        if (null == this.optionProEnumSet) {
            this.optionProEnumSet = EnumUtil.buildEnumSet(this.getValue(), OptionProjectEnum.class);
        }
        return optionProEnumSet;
    }

    public boolean disableUser() {
        return this.containsOption(OptionProjectEnum.DISABLE_USER);
    }

    public boolean muteUser() {
        return this.containsOption(OptionProjectEnum.MUTE_USER);
    }

    public boolean activeUser() {
        return this.containsOption(OptionProjectEnum.ACTIVE_USER);
    }

    private boolean containsOption(OptionProjectEnum optionProjectEnum) {
        return this.retriveOptionProjectEnumSet().contains(optionProjectEnum);
    }

    public enum OptionProjectEnum implements EnumIndex {
        DISABLE_USER(0), MUTE_USER(1), ACTIVE_USER(2);

        private final int bitIndex;

        OptionProjectEnum(int bitIndex) {
            this.bitIndex = bitIndex;
        }

        public int getBitIndex() {
            return bitIndex;
        }
    }
}
```

在上述的EnumSet中，定义了变量，但是没有让其序列化，只让value序列化和反序列化。这种方式主要是采用懒汉式实现加载EnumSet，在我们通过接口获取缓存时，如果没有用到对于option的判断，我们就不需要去加载EnumSet, 因为通过int--->EnumSet是要经历一次循环操作的，这种不必要的消耗该省则省。当然如果存在对optiona的判断，也就是存在对disableUser(),muteUser(),activeUser()的调用，此时去加载EnumSet用于判断，大大降低性能消耗。

## 5.总结

**基础应用和阶级应用的对比**：

<table>
<tbody>
<tr>
<td></td>
<td>优势</td>
<td>劣势</td>
</tr>

<tr>
<td>基础应用</td>
<td>易于理解，序列化反序列方便，属性显示结果直观可见。</td>
<td>新增option时，代码改动大，不易于维护
</td>
</tr>

<tr>
<td>进阶应用</td>
<td>新增option时，代码改动较小，增加枚举即可。</td>
<td>反序列化复杂度高，有层循环，并且序列化后是一个数据，不直观。</td>
</tr>
</tbody>
</table>

建议大家在存储到缓存中不要存储枚举或者bit的具体index集合，因为这样会导致在反序列化是发生嵌套循环，性能较差；可以直接存储一个int的值到缓存中，因此在反序列的时候根据这个值单层循环来build出枚举集合；当然也可以使用在对象中存储boolean属性的方式，这种方式就是序列化的时候根据枚举集合去序列化对象中boolean属性，反序列的时候直接根据缓存中的属性值赋值即可，这种不会存在反序列化时候出现单层循环。

