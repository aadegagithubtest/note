* 依赖分析
* 动态导入
* 混合自定义模块加载器
* 监听资源变化
* hmr
* 重复资源打包
* 缓存
* source map

### watch
#### 监控文件变化的包
* chokidar

#### 底层实现
* fs.watch
* 校验md5
* 添加延迟
> [fs.watch不靠谱](!https://segmentfault.com/a/1190000015159683)

#### watcher监控流程
- [x] 判断是否开启监控
- [x] 判断父文件是否已被监控
- [x] 删除文件中包含子文件的监控
- [x] 添加文件到监控

#### Bunder打包流程
* watch部分
``` javascript
  async watch(path, asset) {
      this.watcher.watch(path);
      this.watchedAssets.set(path, new Set());
  }
```

* start部分
``` javascript
async start() {
    if (this.options.watch) {
      this.watcher = new Watcher();
      await new Promise(resolve => this.watcher.once('ready', resolve));
      this.watcher.on('change', this.onChange.bind(this));
    }
  }
```

### 
