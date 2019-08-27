---
title: JAVA8-让代码更优雅之List排序
date: 2019-02-27 14:31:12
tags: JAVA

---

```


Collections.sort(list, Comparator.comparing(object :: getId));

先定义一个实体类
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Human {

private String name;
private int age;

}
下面的操作都基于这个类来进行操作。这里面使用了Lombok类库，它用注解的方式实现了基本的get和set等方法，让代码看起来更加的优雅。
JAVA8之前的List排序操作
在Java8之前，对集合排序只能创建一个匿名内部类
new Comparator<Human>() {
@Override
public int compare(Human h1, Human h2) {
return h1.getName().compareTo(h2.getName());
}
}

下面是简单的对Humans进行排序（按名称正序）
@Test
public void testSortByName_with_plain_java() throws Exception {

ArrayList<Human> humans = Lists.newArrayList(
new Human("tomy", 22),
new Human("li", 25)
);

Collections.sort(humans, new Comparator<Human>() {

public int compare(Human h1, Human h2) {
return h1.getName().compareTo(h2.getName());
}
});

Assert.assertThat(humans.get(0), equalTo(new Human("li", 25)));
}
使用Lambda的List排序
使用JAVA8函数式方式的比较器
(Human h1, Human h2) -> h1.getName().compareTo(h2.getName())
下面是使用JAVA8函数式的比较的例子
@Test
public void testSortByName_with_lambda() throws Exception {

ArrayList<Human> humans = Lists.newArrayList(
new Human("tomy", 22),
new Human("li", 25)
);
humans.sort((Human h1, Human h2) -> h1.getName().compareTo(h2.getName()));

Assert.assertThat("tomy", equalTo(humans.get(1).getName()));

}

没有类型定义的排序
对于上面的表达式还可以进行简化，JAVA编译器可以根据上下文推测出排序的类型：
(h1, h2) -> h1.getName().compareTo(h2.getName())
简化后的比较器是这样的:
@Test
public void testSortByNameSimplify_with_lambda() throws Exception {

ArrayList<Human> humans = Lists.newArrayList(
new Human("tomy", 22),
new Human("li", 25)
);
humans.sort((h1, h2) -> h1.getName().compareTo(h2.getName()));

Assert.assertThat("tomy", equalTo(humans.get(1).getName()));

} 

使用静态方法引用
JAVA8还可以提供使用Lambda表达式的静态类型引用，我们在Human类增加一个静态比较方法，如下：
public static int compareByNameThenAge(Human h1, Human h2) {

if (h1.getName().equals(h2.getName())) {
return Integer.compare(h1.ge
```





