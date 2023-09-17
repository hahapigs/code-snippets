## Mybatis-plus 动态排序

#### 方式一：

``` java
@Autowired
private UserRepository userRepository;

public List<User> getUsersBySort(String... sortFields) {
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    
    // Perform sorting based on multiple fields
    for (String sortField : sortFields) {
        if (sortField.startsWith("-")) {
            queryWrapper.orderByDesc(sortField.substring(1));
        } else {
            queryWrapper.orderByAsc(sortField);
        }
    }
    
    // Perform the query with sorting
    return userRepository.selectList(queryWrapper);
}
```

``` java
List<User> sortedUsers = getUsersBySort("lastName", "-age");
```

#### 方式二：

``` java

```

