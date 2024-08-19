### gazelle项目中poll的调用链

无论是项目中涉及的 `poll` 或者是 `__warp_poll` 都会抽象成 `do_poll` 这个函数
```c
int32_t poll(struct pollfd *fds, nfds_t nfds, int32_t timeout)
{
    return do_poll(fds, nfds, timeout);
}
```
`do_poll` 函数实现如下
```c
static int32_t do_poll(struct pollfd *fds, nfds_t nfds, int32_t timeout)
{
    if ((select_posix_path() == PATH_KERNEL) || fds == NULL || nfds == 0) {
        return posix_api->poll_fn(fds, nfds, timeout);
    }

    return g_wrap_api->poll_fn(fds, nfds, timeout);
}
```
`do_poll` 函数根据 `select_posix_path` 的返回值来选择是使用标准的 `POSIX poll` 实现，还是使用一个封装或自定义的 `poll` 实现。其中 `posix_api` 结构体封装了标准库的 `poll`函数，并通过函数指针的方式来调用具体的实现函数。

#### 自定义的 `poll` 函数
```c
g_wrap_api->poll_fn(fds, nfds, timeout);
```

`g_wrap_api` 将 `poll` 注册为两种模式分别是 `rtc` 和 `rtw`。

##### `rtc` 模式下（Real-Time Communication）实时网络通信
```c
g_wrap_api->poll_fn          = rtc_poll;
```
```c
int rtc_poll(struct pollfd *fds, nfds_t nfds, int timeout)
{
    LSTACK_LOG(ERR, LSTACK, "rtc_poll: rtc currently does not support poll\n");
    return -1;
}
```
`rtc` 模式下不支持 `poll` 接口，直接打日志到 `lstack` 当中，并返回错误。

##### `rtw` 模式下（Real-Time Web）优化用于处理 Web 请求或其他需要稳定吞吐量的场景
```c
g_wrap_api->poll_fn          = rtw_poll;
```
```c
int rtw_poll(struct pollfd *fds, nfds_t nfds, int timeout)
{
    return lstack_poll(fds, nfds, timeout);
}
```
`rtw_poll` 正式调用 `lstack_poll` 函数，至此调用链完成。

