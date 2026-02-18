# 图标库使用指南

## 已安装的图标库

项目已引入 **Iconify** 图标框架，包含以下图标集：

1. **Ant Design Icons** - 阿里巴巴设计规范图标
2. **Heroicons** - Tailwind CSS 官方图标

## 使用方法

### 1. 基础用法

```vue
<template>
  <Icon icon="ant-design:home-outlined" :width="24" :height="24" />
  <Icon icon="heroicons:home-20-solid" :width="24" :height="24" />
</template>
```

### 2. 自定义颜色和大小

```vue
<Icon 
  icon="ant-design:check-circle-filled" 
  :width="32" 
  :height="32" 
  color="#52c41a" 
/>
```

### 3. 添加动画

```vue
<Icon 
  icon="ant-design:loading-outlined" 
  :width="24" 
  :height="24" 
  class="icon-spin" 
/>

<style>
.icon-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
```

## 常用图标名称

### Ant Design Icons

- 首页：`ant-design:home-outlined` / `ant-design:home-filled`
- 用户：`ant-design:user-outlined` / `ant-design:user`
- 设置：`ant-design:setting-outlined` / `ant-design:setting-filled`
- 成功：`ant-design:check-circle-outlined` / `ant-design:check-circle-filled`
- 失败：`ant-design:close-circle-outlined` / `ant-design:close-circle-filled`
- 加载：`ant-design:loading-outlined`
- 搜索：`ant-design:search-outlined`
- 编辑：`ant-design:edit-outlined`
- 删除：`ant-design:delete-outlined`
- 更多：`ant-design:more-outlined`

### Heroicons

- 首页：`heroicons:home-20-solid`
- 用户：`heroicons:user-20-solid`
- 设置：`heroicons:cog-6-tooth-20-solid`
- 成功：`heroicons:check-circle-20-solid`
- 失败：`heroicons:x-circle-20-solid`
- 刷新：`heroicons:arrow-path-20-solid`
- 搜索：`heroicons:magnifying-glass-20-solid`
- 编辑：`heroicons:pencil-20-solid`
- 删除：`heroicons:trash-20-solid`
- 更多：`heroicons:ellipsis-horizontal-20-solid`

## 查找更多图标

访问以下网站浏览完整图标库：

- **Ant Design Icons**: https://icon-sets.iconify.design/ant-design/
- **Heroicons**: https://icon-sets.iconify.design/heroicons/
- **所有图标**: https://icon-sets.iconify.design/

## 许可证

- **Ant Design Icons**: MIT License
- **Heroicons**: MIT License
- **Iconify**: MIT License

所有图标均可免费商用。

## 注意事项

1. Icon 组件已在 `main.js` 全局注册，无需导入
2. 图标名称格式：`图标集:图标名`
3. 颜色属性接受任何有效的 CSS 颜色值
4. 尺寸单位为像素（px）
