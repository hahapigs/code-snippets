## Mybatis-plus 多字段动态排序

#### 方式一：根据特殊符号实现多字段排序（单表）

``` java
@Autowired
private UserRepository userRepository;

public IPage<User> getUsersBySort(String... sortFields) {
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
    return userRepository.selectPage(new Page(1, 10), queryWrapper);
}
```

``` java
IPage<User> sortedUsers = getUsersBySort("lastName", "-age");
```

#### 方式二：根据 Page 关键字实现多字段排序（单表）

``` java
		@Autowired
    private UserMapper userMapper;

    /**
     * 分页
     *
     * @param query
     * @return
     */
    @Override
    public IPage<UserPO> selectPage(UserQuery query) {
        LambdaQueryWrapper<UserPO> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.and(StringUtils.hasText(query.getKeywords()), wrapper -> wrapper.like(UserPO::getAccount, query.getKeywords())
                .or().like(UserPO::getNickName, query.getKeywords())
                .or().like(UserPO::getRealName, query.getKeywords()));
      	Page<UserPO> page = new Page<>(1, 10);
        page.addOrder(OrderItem.desc("create_time");
        return userMapper.selectPage(page, queryWrapper);
    }
```

#### 方式三：自定义排序集合实现多字段排序（连表）

``` java
/**
 * @since 1.0
 */
public class OrderUtils {
		/**
     * 获取排序规则
     *
     * @param clazz
     * @return map
     */
    @Deprecated(since = "1.1", forRemoval = true)
    public Map<String, String> getOrderItemMap(Class<? extends Serializable> clazz) {
        Map<String, String> orderItemMap = new LinkedHashMap<>();
        if (!CollectionUtils.isEmpty(orders)) {
            orders.forEach(item -> {
                try {
                    clazz.getDeclaredField(item.getColumn());
                    orderItemMap.put(com.example.canary.util.StringUtils.toUnderlineCase(item.getColumn()), item.isAsc() ? "asc" : "desc");
                } catch (NoSuchFieldException e) {
                    throw new IllegalArgumentException(item.getColumn() + "不符合列名的要求");
                }
            });
        }
        return orderItemMap;
    }

    /**
     * 获取排序规则
     *
     * @param clazz
     * @return
     */
    public Map<String, String> getOrderMap(Class<?> clazz) {
        // 判空
        if (CollectionUtils.isEmpty(orders)) {
            return Collections.emptyMap();
        }

        Map<String, String> orderItemMap = new LinkedHashMap<>();
        List<String> orderFeildList = orders.stream().map(OrderItem::getColumn).toList();
        List<Field> fields = new ArrayList<>(Arrays.asList(clazz.getDeclaredFields()));
        if (clazz.getSuperclass() != null) {
            List<Field> superFields = Arrays.asList(clazz.getSuperclass().getDeclaredFields());
            fields = Stream.concat(fields.stream(), superFields.stream()).toList();
        }
        Set<String> fieldSet = new HashSet<>();
        fields.forEach(field -> {
            String fieldName = field.getName();
            fieldSet.add(fieldName);
        });
        String errorFiled = orderFeildList.stream().filter(e -> !fieldSet.contains(e)).findFirst().orElse(null);
        if (StringUtils.hasText(errorFiled)) {
            throw new IllegalArgumentException(errorFiled + " 不符合列名的要求");
        }
        fields.forEach(field -> {
            String fieldName = field.getName();
            if (StringUtils.hasText(fieldName)) {
                orders.forEach(item -> {
                    if (item.getColumn().equals(fieldName)) {
                        String cloumn = null;
                        if (field.isAnnotationPresent(TableField.class)) {
                            TableField tableField = field.getAnnotation(TableField.class);
                            cloumn = tableField.value();
                            if (!StringUtils.hasText(cloumn)) {
                                cloumn = com.example.canary.util.StringUtils.toUnderlineCase(cloumn);
                            }
                        } else {
                            cloumn = com.example.canary.util.StringUtils.toUnderlineCase(fieldName);
                        }
                        orderItemMap.put(cloumn, item.isAsc() ? "asc" : "desc");
                    }
                });
            }
        });
        return orderItemMap;
    }
}
```

```java
		@Autowired
    private MenuMapper menuMapper;

    /**
     * pages
     *
     * @param query
     * @return
     */
    @Override
    public IPage<MenuVO> selectPageVo(MenuQuery query) {
      	Page<UserPO> page = new Page<>(1, 10);
        List<OrderItem> orders = new ArrayList<>();
        orders.add(OrderItem.desc("create_time"));
      	orders.add(OrderItem.asc("name"));
        return menuMapper.selectPageVo(page, query, OrderUtil.getOrderMap(MenuVO.class));
    }
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.canary.sys.mapper.MenuMapper">


    <select id="selectPageVo" resultType="com.example.canary.sys.entity.MenuVO">
        select
            m.*
        from
            sys_menu m
        where
            m.is_deleted = 0
        <if test="query.keywords != null and query.keywords != ''">
            and m.`name` like concat('%', #{query.keywords}, '%')
        </if>
        <if test="orderMap != null and orderMap.size > 0">
            order by
            <foreach collection="orderMap.entrySet()" item="value" index="key" open="" separator="," close="">
                ${key} ${value}
            </foreach>
        </if>
    </select>
</mapper>


```

#### 方式四：根据 Page 关键字实现多字段排序（连表）

``` java
@Data
public class BasePage<T extends Serializable> {


    /**
     * 每页显示条数
     */
    private long size = 500;

    /**
     * 当前页
     */
    private long current = 1;

    /**
     * 排序
     */
    private List<OrderItem> orders;

    /**
     * 是否驼峰
     */
    private boolean camelCase = true;

    /**
     * 获取当前页第一行行号
     *
     * @return
     */
    public long getStartNum() {
        return (current - 1) * size;
    }

    /**
     * 获取当前页面末尾行行号
     *
     * @return
     */
    public long getEndNum() {
        return current * size;
    }


    /**
     * 获取 page
     *
     * @return
     */
    public Page<T> getPage() {
        Page<T> page = new Page<>();
        BeanUtils.copyProperties(this, page);
        if (!CollectionUtils.isEmpty(orders)) {
            orders.forEach(item -> {
                String column = com.example.canary.util.StringUtils.toUnderlineCase(item.getColumn());
                item.setColumn(column);
            });
        }
        return page;
    }
} 

```



``` java
		@Autowired
    private MenuMapper menuMapper;

    /**
     * pages
     *
     * @param query
     * @return
     */
    @Override
    public IPage<MenuVO> selectPageVo(MenuQuery query) {
        return menuMapper.selectPageVo(query.getPage(), query);
    }
```

XML 中的 sql 语句无需考虑分页和多字段动态排序的问题，mybatis-plus 会自动追加