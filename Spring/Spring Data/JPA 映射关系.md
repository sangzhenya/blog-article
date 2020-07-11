---
title: "JPA 映射关系"
tags: ["Spring", "JPA"]
categories: ["Spring"]
date: "2019-01-26T09:00:00+08:00"
---

对于一对多的情况使用一个 Department 类和一个 Student 类。

### 单向多对一

在多的一端（Student）维护数据，则需要在多的一端设置如下的注解：

```java
private Department department;
// 设置多对一，且设置懒加载
@ManyToOne(fetch = FetchType.LAZY)
// 设置 Mapping 列的名称
@JoinColumn(name = "department_id")
public Department getDepartment() {
  return department;
}

public void setDepartment(Department department) {
  this.department = department;
}
```

在插入的时候最好先插入一的一端，再插入多的一端。如果反之则会多出数条 Update 语句，因为插入多的一端的时候还不知道一的一端对应的值，所以需要数条 Update 语句在插入一的一端后更新多的一端。

### 单向一对多

在一的一端（Department）维护数据，则需要在一的一端设置如下注解：

```java
private Set<Student> students;
// 设置一对多，且懒加载
@OneToMany(fetch = FetchType.LAZY)
// 设置 Mapping 列的名称
@JoinColumn(name = "department_id")
public Set<Student> getStudents() {
  return students;
}
public void setStudents(Set<Student> students) {
  this.students = students;
}
```

默认如果删除 一的一端，则会把关联的多的一端的外键置空，然后删除。可以通过修改 OneToMany 的属性修改 `@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.REMOVE)` 例如修改为 REMOVE，则会将多的一端全部删除后再删除一的一端，即级联删除。

### 双向一对多

在一的一端和多的一端均维护关系，多的一端和上面设置的相同，一的一端需要设置交给多的一端管理。设置如下：

```java
private Set<Student> students;
// 设置一对多，且将管理交给一的一端，且不需要设置 @JoinColumn
@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.REMOVE,
            targetEntity = Student.class, mappedBy = "department")
public Set<Student> getStudents() {
  return students;
}
public void setStudents(Set<Student> students) {
  this.students = students;
}
```

### 双向一对一

在双向一对一的关系中使用 User 和 UserInfo 类。

```java
// User
private UserInfo userInfo;
// 设置一对一，且将关联交给 UserInfo
@OneToOne(mappedBy = "user")
public UserInfo getUserInfo() {
  return userInfo;
}
```

```java
// UserInfo
private User user;
// 设置一对一，且设置懒加载
@OneToOne(fetch = FetchType.LAZY)
// 设置维护关系的列的名称
@JoinColumn(name = "user_id", unique = true)
public User getUser() {
  return user;
}
public void setUser(User user) {
  this.user = user;
}
```

对于双向一对一的关联关系，先保存不维护关系的一方，否则还是会多出一条 Update 更新语句。

### 双向多对多

在双向多对多的关系中使用 Article 和 Tag 类。需要指定一个关系维护端，可以通过在 @ManyToMany 的属性 mappedBy 来标识关系维护端。

```java
// Tag
private Set<Article> articles;
// 设置多对多，且将维护关系交给 Article
@ManyToMany(mappedBy = "tags")
public Set<Article> getArticles() {
  return articles;
}
public void setArticles(Set<Article> articles) {
  this.articles = articles;
}
```

```java
// Article
private Set<Tag> tags;
// 设置多对多
@ManyToMany
// 设置中间表
@JoinTable(
  // 中间表名称
  name = "book_tag",
  // name 是中间表列的名称，referencedColumnName 中间表关联 Article 表的列名
  joinColumns = {@JoinColumn(name = "article_id", referencedColumnName = "id")},
  // name 是中间表列的名称，referencedColumnName 中间表关联  Tag 表的列名
  inverseJoinColumns = {@JoinColumn(name = "tag_id", referencedColumnName = "id")}
)
public Set<Tag> getTags() {
  return tags;
}

public void setTags(Set<Tag> tags) {
  this.tags = tags;
}
```

