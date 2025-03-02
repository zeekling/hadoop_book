
# 简介

container-executor 是NodeManager管理Container很重要的一个工具，是深入学习Yarn作业调度不可缺少的一个知识点，值得深入学习。本文只描述比较基础的功能点(目前不会包含Docker相关)。


# 核心功能点

## checksetup
主要是检查container-executor的配置是否ok，没有其他功能。核心代码如下：  

```c
case CHECK_SETUP:
  //we already did this 
  exit_code = 0;
  break;
```

## mount-cgroups
在配置项feature.mount-cgroup.enabled为true的时候为nodemanager挂载cgroup。核心是调用系统函数`mount`。下面代码中的是配置的挂载点。由命令行参数传入。

```c
if (mount("none", mount_path, "cgroup", 0, controller) == 0) {
  // 挂载成功
    if (mkdirs(hier_path, perms) == 0) {
        change_owner(hier_path, nm_uid, nm_gid);
        // 修改子目录权限。
        chown_dir_contents(hier_path, nm_uid, nm_gid);
    }
}
```

## exec-container

**前提条件**：配置`feature.terminal.enabled=true`

当前功能的核心实现在`container-executor.c`的函数`int exec_container(const char *command_file)`中。
在非Docker模式下，主要步骤如下：
```c
// 切换用户
if (change_user(user_detail->pw_uid, user_detail->pw_gid) != 0) {
  _exit(DOCKER_EXEC_FAILED);
}
// 切换工作目录
ret = chdir(workdir);
if (ret != 0) {
  fprintf(ERRORFILE, "chdir failed - %s", strerror(errno));
  _exit(DOCKER_EXEC_FAILED);
}
// 执行启动脚本。
execve(binary, args, env);
fprintf(ERRORFILE, "exec failed - %s\n", strerror(errno));
_exit(DOCKER_EXEC_FAILED);
```
最后会执行配置launch-command中的命令。当前步骤的核心应该主要是判断当前用户是否有权限启动Container。


## 启动Container 

真正启动Container，参数格式如下：

`container-executor <user> <yarn-user> <command> <command-args>`
源代码中的解释如下：
```c  
fprintf(stream,
    "       container-executor <user> <yarn-user> <command> <command-args>\n"
    "       where command and command-args: \n" \
    "            initialize container:  %2d appid containerid tokens nm-local-dirs "
    "nm-log-dirs cmd...\n"
    "            launch container:      %2d appid containerid workdir "
    "container-script tokens http-option pidfile nm-local-dirs nm-log-dirs resources ",
    INITIALIZE_CONTAINER, LAUNCH_CONTAINER);
```

可以看出提供了两个功能：
- 初始化Container。
- 启动Container。

