---
title: 权限管理（五）
date: 2022-03-13 17:11:52
categories:
- web
tags:
- web
- 权限管理
- vue antd admin
- ruoyi
---

# 菜单管理
主要功能为：查看菜单列表、添加菜单、修改菜单、删除菜单
## 后端实现
### 查看菜单列表
1. MenuController  
``` java
@PostMapping("/tree")
public ResponseData menuTree(@RequestBody(required = false) Menu menu){
    Long userId = BaseController.getUserId();
    List<Menu> menus = menuService.selectMenuList(menu, userId);
    List<Menu> menuTree = menuService.buildMenuTree(menus);
    return ResponseData.success(menuTree);
}
```

### 添加菜单
1. MenuController  
``` java
@PostMapping
public ResponseData add(@RequestBody Menu menu) {
    if (menu.getName() == null){
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    // 检查菜单名称
    if (!menuService.checkMenuNameUnique(menu)) {
        return ResponseData.setResult(ResultCodeEnum.MENU_HAS_EXIST);
    }
    // 添加菜单
    menu.setCreatedBy(BaseController.getUserName());
    int result = menuService.insertMenu(menu);
    if (result > 0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_ADD_FAILURE);
    }
}
```
2. MenuService  
``` java
public int insertMenu(Menu menu) {
    return menuDao.insert(menu);
}
```

### 修改菜单
1. MenuController  
``` java
@PutMapping()
public ResponseData update(@RequestBody Menu menu) {
    if (menu.getId() == null || menu.getId().equals(menu.getPId())) {
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    // 修改菜单
    menu.setUpdatedBy(BaseController.getUserName());
    int result = menuService.updateMenu(menu);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_UPDATE_ERROR);
    }
}
```

2. MenuService  
``` java
public int updateMenu(Menu menu) {
    return menuDao.updateById(menu);
}
```

### 删除菜单
1. MenuController  
``` java
@DeleteMapping("/{menuId}")
public ResponseData delete(@PathVariable("menuId") Long menuId) {
    if (menuId == null) {
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    if (menuService.hasChildByMenuId(menuId)) {
        return ResponseData.setResult(ResultCodeEnum.SUB_MENU_HAS_EXIST);
    }
    if (menuService.checkMenuExistRole(menuId)) {
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    // 删除菜单
    int result = menuService.deleteMenuById(menuId);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.MENU_ROLE_HAS_EXIST);
    }

}
```

2. MenuService  
``` java
public int deleteMenuById(Long menuId) {
    return menuDao.deleteById(menuId);
}
```

## 前端
![20220313172946.png](https://s2.loli.net/2022/03/13/COKElZLQUVhwI2g.png)
![20220313173016.png](https://s2.loli.net/2022/03/13/WLsyZdTNlcekI4P.png)
![20220313173033.png](https://s2.loli.net/2022/03/13/MuYCX7tJGKhd8mb.png)