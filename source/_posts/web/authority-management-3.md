---
title: 权限管理（三）
date: 2022-03-13 16:32:13
categories:
- web
tags:
- web
- 权限管理
- vue antd admin
- ruoyi
---
# 用户管理
主要功能为：查看用户列表、获取用户详情、注册用户、修改用户、删除用户
## 后端实现
### 查看用户列表
1. UserController  
``` java
@PostMapping("/list") 
public ResponseData list(@RequestBody(required = false) User user,
                                @ApiParam("页数") @RequestParam(defaultValue = "1", value = "pageNum") Integer pageNum,
                                @ApiParam("每页的数据数量") @RequestParam(defaultValue = "10", value = "pageSize") Integer pageSize){
    PageData<User> userList = userService.selectUserList(new Page<>(pageNum, pageSize), user);
    return ResponseData.success(userList);
}
```
2. UserService  
这里查询数据，用的是mybatis plus。分页，也用的是mybatis plus（需要配置，开启分页）
``` java
public PageData<User> selectUserList(Page<User> page, User user) {
    QueryWrapper<User> wrapperUser = new QueryWrapper<>();
    if (user != null) {
        wrapperUser.eq( user.getId() != null,"id",user.getId());
        wrapperUser.eq( user.getName() != null,"name",user.getName());
        wrapperUser.eq( user.getIsDeleted() != null,"is_deleted",user.getIsDeleted());
    }
    Page<User> userPage = userDao.selectPage(page, wrapperUser);
    return new PageData<>(userPage);
}
```

### 获取用户详情
1. UserController   
这个接口，主要是给前端的。用户详情返回用户拥有的角色，和角色列表。  
这里有一个逻辑处理，就是，如果当前用户是超级管理员，他是可以看到超级管理员的角色的。如果当前用户角色不是超级管理员，那么它就看不到超级管理员（非超级管理员没有权利授权超级管理员）  
这个接口，有两个Mapping值，是因为添加新用户和查看某个用户详情的时候，都需要查看用户角色。添加新用户，只需要看当前用户能查看到的角色列表，而查看用户详情，还需要查看当前用户所拥有的角色。这里将这两个接口，写到一起了。
``` java
@GetMapping(value = {  "/","{userId}" })
    public ResponseData getUserInfo(@PathVariable(value = "userId",required = false) Long userId){
    // 1. 检查当前用户是否有访问权限？略
    HashMap<String,Object> data = new HashMap<>();
    // 3. 查询所有角色
    List<RoleVo> roles = roleService.selectRoleAll();
    // 2. 查询用户当前拥有的角色
    if (userId != null) {
        List<Long> roleIds = roleService.selectRoleListByUserId(userId);
        data.put("roleIds",roleIds);
        if (!roleIds.contains(1L)) {// 如果当前用户不是超级管理员，则其看不到超级管理员的角色
            roles = roles.stream().filter(r->r.getId()!=1L).collect(Collectors.toList());
        }
    } else {
        System.out.println(BaseController.getRoles());
        if (!BaseController.getRoles().contains("SUPER_ADMIN")) {// 如果当前用户不是超级管理员，则其看不到超级管理员的角色
            roles = roles.stream().filter(r->r.getId()!=1L).collect(Collectors.toList());
        }
    }
    data.put("roles",roles);
    return ResponseData.success(data);
}
```
2. RoleService  
因为都是根据用户Id查询角色，所以调的都是RoleService。

查询所有角色
``` java
@Override
public List<RoleVo> selectRoleAll() {
    QueryWrapper<Role> roleQueryWrapper = new QueryWrapper<>();
    roleQueryWrapper.eq("is_deleted",false);
    List<Role> roleList = roleDao.selectList(roleQueryWrapper);
    return roleList.stream().map(RoleVo::new).collect(Collectors.toList());
}
```

根据用户ID查询当前拥有的角色Id
``` java
public List<Long> selectRoleListByUserId(Long userId) {
    return roleDao.selectRoleListByUserId(userId);
}
```

3. RoleMapper  
``` xml
<select id="selectRoleListByUserId" resultType="java.lang.Long">
    select distinct r.id
    from role r
                left join user_role ur on ur.role_id = r.id
                left join user u on u.id = ur.user_id
    where r.is_deleted = 0 and ur.user_id = #{userId}
</select>
```

### 添加用户
1. UserController  
``` java
@PostMapping("")
public ResponseData add(@RequestBody User user){
    if (user.getName() == null || user.getPassword()== null){
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    // 判断用户名是否唯一
    if (!userService.checkUserNameUnique(user.getName())) {
        return ResponseData.setResult(ResultCodeEnum.USER_HAS_EXIST);
    }
    // 设置当前用户的被添加人
    user.setCreatedBy(BaseController.getUserName());
    // 添加用户
    int result = userService.register(user);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_ADD_FAILURE);
    }
}
```
2. UserService  
有一个特别的，就是添加用户的时候，也会添加用户角色。
``` java
public int register(User user) {
    List<User> users = userDao.selectUserByUserName(user.getName());
    if (users.size()> 0){
        throw new WebException(ResultCodeEnum.USER_HAS_EXIST);
    }
    // 用户密码加密
    user.setPassword(new BCryptPasswordEncoder().encode(user.getPassword()));
    int rows = userDao.insert(user);
    Option.of(rows).getOrElseThrow(() -> new WebException(ResultCodeEnum.DB_ADD_FAILURE));
    // 添加用户角色
    insertUserRole(user);
    return rows;
}

public void insertUserRole(User user) {
    List<Long> roleIds = user.getRoleIds();
    if (roleIds != null) {
        roleIds.forEach(roleId -> {
            UserRole ur = new UserRole();
            ur.setUserId(user.getId());
            ur.setRoleId(roleId);
            userRoleDao.insert(ur);
        });

    }
}
```

### 修改用户
1. UserController  
``` java
@PutMapping()
public ResponseData update(@RequestBody User user){
    if (user.getId() == null){
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    user.setPassword(null);
    // 检查修改权限，略
    user.setUpdatedBy(BaseController.getUserName());
    int result = userService.update(user);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_UPDATE_ERROR);
    }
}
```

2. UserService  
这里，修改用户时，如果传了角色Id，则先删除用户关联的角色Id，再新添加
``` java
public int update(User user) {
    if (user.getPassword() != null) {
        user.setPassword(new BCryptPasswordEncoder().encode(user.getPassword()));
    }
    if (user.getRoleIds() != null) {
        // 删除用户与角色关联
        deleteUserRoleByUserId(user.getId());
        // 重新添加用户与角色关联
        insertUserRole(user);
    }
    // 更新用户
    return Option.of(userDao.updateById(user))
            .getOrElseThrow(() -> new WebException(ResultCodeEnum.DB_UPDATE_ERROR));
}

public void deleteUserRoleByUserId(Long id) {
    QueryWrapper<UserRole> wrapperUser = new QueryWrapper<>();
    wrapperUser.eq("user_id",id);
    userRoleDao.delete(wrapperUser);
}
```

### 删除用户
1. UserController  
``` java
@DeleteMapping("/{id}")
public ResponseData delete(@PathVariable("id") Long id){
    if (id == null){
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    if (id.equals(BaseController.getUserId())) {
        return ResponseData.setResult("201","当前用户不允许删除");
    }
    // 判断，不能删除自己
    int result = userService.delete(id);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_DELETE_FAILURE);
    }
}
```

2. UserService  
删除用户，会删除相关联的角色Id
``` java
public int delete(Long id) {
    // 检查有没有删除用户的权限
    // 删除用户角色
    deleteUserRoleByUserId(id);
    // 删除用户
    return Option.of(userDao.deleteUserById(id))
            .getOrElseThrow(() -> new WebException(ResultCodeEnum.DB_UPDATE_ERROR));
}
```

## 前端
基本布局就是一个table，展示用户列表，然后详情显示用户详情（这里包含了用户的角色）。
![1647162520.jpg](https://s2.loli.net/2022/03/13/hBYTxbfc6e1A2jR.png)
![1647162565.jpg](https://s2.loli.net/2022/03/13/ATPwQtGjCDWry47.png)
基本上就是写请求，拿数据，展示啥的，就不详细写了。