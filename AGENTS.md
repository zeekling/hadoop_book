# AGENTS.md

## 仓库
Apache Hadoop 中文技术文档仓库，分析组件源码和架构模式。

## 结构
```
hadoop_book/
├── common/   # 身份验证、Fair Call Queue、NativeIO
├── hdfs/     # Namenode、DataNode、FsImage、JournalNode
├── yarn/     # ResourceManager、调度器、Container
├── zookeeper/
├── ozone/
└── watch/    # 特性追踪
```

## Git（关键：禁止推 master）
```bash
# 创建功能分支
git checkout -b feature/your-feature

# 提交后推送远程
git push origin feature/your-feature
```
本地有 pre-push hook 阻止推 master。

## 文件命名
- 模块 README：`{模块名}/README.md`
- 技术详解：`{模块}/{组件}详解.md`
- kebab-case：`fair_call_queue.md`

## 内容结构
1. 标题（# 标题）
2. 简介
3. 技术细节和代码示例
4. 架构图：`![pic](https://pan.zeekling.cn/zeekling/hadoop/...)`
5. 比较表格

## 代码规范
- 类：`org.apache.hadoop.hdfs.server.namenode.FSImage`
- 方法：`ClassName#methodName`
- 配置：`` `dfs namenode.name.dir` ``
- 版本：3.3.1（不用 v3.3.1）

## 编译命令
- Linux：`mvn -T 8 package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true`
- Windows：`mvn -T 1C clean install -DskipTests -PskipShade -P\!native-win -Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dpmd.skip=true`

## 关键规则
- 所有内容中文
- 示例用 Java
- 图片用 pan.zeekling.cn/zeekling/hadoop/
- 链接内部用相对路径
