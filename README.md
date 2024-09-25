
# 微博同步关注列表

> A：待同步账号。B：需要同步的账号

## 起因

由于微博账号因不当转发被封号，需要将 A 账号的关注列表同步到新的 B 账号。

## 思路

通过比对 A 账号与 B 账号的关注列表，筛选出 B 账号未关注的用户，并自动执行关注操作。

## 修改点

本项目 forked from `lxw15337674/weiboSyncFollow`（[项目地址](https://github.com/lxw15337674/weiboSyncFollow)），增加了关注时的随机等待时间，以防止被微博服务器限流封禁。被封禁后无法搜索到被封账号，故删除了方案 1。使用 Chrome 浏览器时，需在 Chrome 根目录下运行以下命令：

```bash
chrome.exe --disable-web-security --user-data-dir="C:/ChromeDev"
```

## 中间的坑点

### 1. 关注列表限制

由于微博的自我保护策略，可能只能看到部分关注列表。为解决此问题，需登录待同步账号（A 账号）以获取完整的关注列表。

### 2. 微博关注接口并发限制

微博 API 对于频繁操作有严格的并发限制，因此需要设置请求延时。频率过快将触发微博的限流策略，导致关注失败。

### 3. 每日最大关注数量限制

微博每天允许的最大关注数量约为 150 人，超过此数后，每次关注需要输入验证码才能继续。

### 4. 关注接口的请求头

新版微博的关注接口 `https://www.weibo.com/ajax/friendships/create` 需要在请求头中加入 `xsrf token`，否则会报错 403。旧版接口 `https://weibo.com/aj/f/followed` 则不需要该 token。

## 操作步骤

### 1. CMD 命令配置 Chrome

首先，在 CMD 中输入以下命令以关闭 Chrome 的安全模式：

```bash
chrome.exe --disable-web-security --user-data-dir="C:/ChromeDev"
```

### 2. 获取 A 账号关注列表

1. 前往 [微博首页](https://www.weibo.com/)，登录 A 账号。
2. 打开开发者工具 (`F12`)，在控制台中粘贴以下代码并执行，等待提示“获取列表成功”：

   ```javascript
   const key = 'followList';
   const getOwnFollowed = async (page = 1, list = []) => {
     const res = await fetch(`/ajax/profile/followContent?page=${page}`, {
       method: 'GET',
       headers: {
         "content-type": 'application/json'
       }
     }).then(res => res.json());
     const users = res.data.follows.users;
     if (users.length > 0) {
       list.push(...users);
       await getOwnFollowed(++page, list);
     }
     return list;
   }
   const start = async () => {
     const list = await getOwnFollowed();
     localStorage.setItem('followList', JSON.stringify(list.map(item => ({ id: item.id }))));
     console.log(`获取列表成功,共${list.length}个`);
   }
   start();
   ```

3. 退出 A 账号，登录 B 账号。

### 3. 获取 `x-xsrf-token`

在开发者工具中，前往 "应用程序" (Application) 标签页，找到 `x-xsrf-token` 的值（如下图所示）：
![csrf-token位置](https://github.com/lxw15337674/weiboSyncFollow/assets/19898669/d5691f35-9d14-41a6-855d-8d6d9de3eecb)

### 4. 执行关注操作

在控制台中粘贴以下代码，替换 `(复制的token)` 为你从上一步中获得的 `x-xsrf-token` 值，执行后等待提示成功：

```javascript
let token = '';
function sleep(time) {
  return new Promise((resolve) => setTimeout(resolve, time));
}

const getOwnFollowed = async (page = 1, list = []) => {
  const res = await fetch(`/ajax/profile/followContent?page=${page}`, {
    method: 'GET',
    headers: {
      "content-type": 'application/json'
    }
  }).then(res => res.json());
  const users = res.data.follows.users;
  if (users.length > 0) {
    list.push(...users);
    await getOwnFollowed(++page, list);
  }
  return list;
}

const getRandomDelay = () => {
  const min = 30000; // 30 seconds
  const max = 120000; // 2 minutes
  const delay = Math.floor(Math.random() * (max - min + 1)) + min;
  console.log(`随机延迟 ${delay / 1000} 秒`);
  return delay;
}

const batchFollow = async (list = [], index = 0) => {
  if (index >= list.length) return; // 结束条件
  const followUser = list[index];
  
  await fetch(`/ajax/friendships/create`, {
    method: 'POST',
    headers: {
      "content-type": 'application/json;charset=UTF-8',
      "accept": "application/json, text/plain, */*",
      "x-xsrf-token": token
    },
    body: JSON.stringify({
      friend_uid: followUser.id,
      lpage: "profile",
      page: "profile"
    })
  }).then(res => res.json()).then(async (res) => {
    console.log(`[${index + 1}/${list.length}], ${res.name}, 关注成功`);
    await sleep(getRandomDelay()); // 等待随机时间
    await batchFollow(list, index + 1); // 递归调用关注下一个用户
  });
}

const start = async (t) => {
  token = t;
  const list = await getOwnFollowed();
  let unFollowList = JSON.parse(localStorage.getItem('followList')).filter(unFollowItem => list.findIndex(item => item.id === unFollowItem.id) === -1);
  console.log(`获取待关注列表成功,共${unFollowList.length}个`);
  await batchFollow(unFollowList);
}

start("(复制的token)"); // 例如：start("ApMq0KHPGo3SBclIGe6dMpn7")
```

---

通过增加随机等待时间以及合理的操作流程，可以有效避免被微博封禁，同时确保关注操作的顺利完成。
```
