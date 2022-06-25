# Mysql 源码阅读（二）登录连接调试
***



持续创作，加速成长！这是我参与「掘金日新计划 · 6 月更文挑战」的第4天，[点击查看活动详情](https://juejin.cn/post/7099702781094674468?utm_source=xitongxiaoxi&utm_medium=push&utm_campaign=kechengfenxiao)



## 简介

在上篇中我们将源码在本地顺利的跑了起来，接下来我们需要连接我们的数据库，为后面的工作做一定的准备



## 使用docker容器尝试连接

我们首先使用docker容器去启动连接我们上篇文章中成功运行起来的MySQL服务器，但没能如愿，没有连接成功

下面的错误，以前我们也是经常见到了

```powershell
PS C:\Users\lw> docker run -ti mysql mysql -h 192.168.1.4 -P 3306 -u root -p
Enter password:
ERROR 1130 (HY000): Host 'DESKTOP-8U69O9P' is not allowed to connect to this MySQL server
```

我们使用： is not allowed 去全局搜索下代码，找到下面的代码，通过打上断点，也走到了这步里面

成功的找到了代码入口

```cpp
bool acl_check_host(const char *host, const char *ip)
{
  mysql_mutex_lock(&acl_cache->lock);
  if (allow_all_hosts)
  {
    
    mysql_mutex_unlock(&acl_cache->lock);
    return 0;
  }

  if ((host && my_hash_search(&acl_check_hosts,(uchar*) host,strlen(host))) ||
      (ip && my_hash_search(&acl_check_hosts,(uchar*) ip, strlen(ip))))
  {
    mysql_mutex_unlock(&acl_cache->lock);
    return 0;                                   // Found host
  }
  for (ACL_HOST_AND_IP *acl= acl_wild_hosts->begin();
       acl != acl_wild_hosts->end(); ++acl)
  {
    if (acl->compare_hostname(host, ip))
    {
      mysql_mutex_unlock(&acl_cache->lock);
      return 0;                                 // Host ok
    }
  }
  mysql_mutex_unlock(&acl_cache->lock);
  if (ip != NULL)
  {
    /* Increment HOST_CACHE.COUNT_HOST_ACL_ERRORS. */
    Host_errors errors;
    errors.m_host_acl= 1;
    inc_host_errors(ip, &errors);
  }
  return 1;                                     // Host is not allowed
}
```

通过调试，我们发现acl_wild_hosts里面没有值导致的，导致ip==null，从而host没有匹配上，不允许连接

我们在当前文件中没有搜索到acl_wild_hosts，只能在整个解决方案中进行搜索，我们定位到下面一段代码：

```cpp
static void init_check_host(void)
{
  DBUG_ENTER("init_check_host");
  if (acl_wild_hosts != NULL)
    acl_wild_hosts->clear();
  else
    acl_wild_hosts=
      new Prealloced_array<ACL_HOST_AND_IP, ACL_PREALLOC_SIZE>(key_memory_acl_mem);

  size_t acl_users_size= acl_users ? acl_users->size() : 0;

  (void) my_hash_init(&acl_check_hosts,system_charset_info,
                      acl_users_size, 0, 0,
                      (my_hash_get_key) check_get_key, 0, 0,
                      key_memory_acl_mem);
  if (acl_users_size && !allow_all_hosts)
  {
    for (ACL_USER *acl_user= acl_users->begin();
         acl_user != acl_users->end(); ++acl_user)
    {
      if (acl_user->host.has_wildcard())
      {                                         // Has wildcard
        ACL_HOST_AND_IP *acl= NULL;
        for (acl= acl_wild_hosts->begin(); acl != acl_wild_hosts->end(); ++acl)
        {                                       // Check if host already exists
          if (!my_strcasecmp(system_charset_info,
                             acl_user->host.get_host(), acl->get_host()))
            break;                              // already stored
        }
        if (acl == acl_wild_hosts->end())       // If new
          acl_wild_hosts->push_back(acl_user->host);
      }
      else if (!my_hash_search(&acl_check_hosts,(uchar*)
                               acl_user->host.get_host(),
                               acl_user->host.get_host_len()))
      {
        if (my_hash_insert(&acl_check_hosts,(uchar*) acl_user))
        {                                       // End of memory
          allow_all_hosts=1;                    // Should never happen
          DBUG_VOID_RETURN;
        }
      }
    }
  }
  acl_wild_hosts->shrink_to_fit();
  freeze_size(&acl_check_hosts.array);
  DBUG_VOID_RETURN;
}
```

从名称和功能上看出，这个就是acl_check_hosts的初始化相关代码

其中acl_users通过断点调试，有三个用户：

- root：我们的经常用到的初始用户
- mysql.session: 这个还真没见过
- mysql.sys: 这个也没有见过

这个函数看下来，大致有两个地方会导致acl_check_hosts的增加：

```cpp
(void) my_hash_init(&acl_check_hosts,system_charset_info,
                      acl_users_size, 0, 0,
                      (my_hash_get_key) check_get_key, 0, 0,
                      key_memory_acl_mem);
```

上面这个是初始化，从里面的参数没有找到什么新增的

下面一段代码是明显新增的，但调试中，代码一直走不进去这段逻辑

```cpp
if (acl == acl_wild_hosts->end())       // If new
          acl_wild_hosts->push_back(acl_user->host);
```

而走不下去的第一步就是

```cpp
if (acl_user->host.has_wildcard())

bool has_wildcard()
  {
    return (strchr(hostname,wild_many) ||
            strchr(hostname,wild_one)  || ip_mask );
  }
```

strchr是一个字符串搜索函数：该函数返回在字符串 str 中第一次出现字符 c 的位置，如果未找到该字符则返回 NULL。

其中的wild_many是我们熟悉的%，而wild_one是"_"，而hostname是localhost，所以没有匹配上

都走不下去，那我们只能看看acl_user的相关代码了

很幸运，在当前文件中，我们找到了acl_users的初始化函数：

```cpp
static my_bool acl_load(THD *thd, TABLE_LIST *tables)

// 函数代码太长了，先粘贴一个关键的
acl_users->push_back(user);
```

而host的初始化在这步：

```cpp
user.host.update_hostname(get_field(&global_acl_memory,
                                      table->field[table_schema->host_idx()]));
```

看着是从表中读取一些数据，有点头大，搞不动了，看不懂（超过认知负载了，我们继续挖下去也挖不动了，只能先往上跳一跳）



## 使用mysql workbench进行连接

为了避免一直卡住不动了，我们还是直接本地安装一个mysql workbench，然后使用localhost的域名和root用户进行连接

如果忘记了初始密码，想要再初始化一次，可能会遇到下面的错误：

```powershell
2022-06-25T00:28:59.791538Z 0 [ERROR] --initialize specified but the data directory has files in it. Aborting.
```

我们把源码根目录下的: release/sql/data 文件夹删除即可

初次链接的使用需要重置密码，重置完成后，就链接上了



然后我们执行下面的脚本，赋予root其他ip的连接权限

```sql
SET SQL_SAFE_UPDATES = 0;
use mysql;
update user set host='%' where user='root';
flush privileges;
```

在我们运行上面的SQL的时候，用户的初始化和host检测的初始化，都重新执行一遍



## 再次使用mysql容器进行连接

然后我们再次运行命令进行连接

```shell
docker run -ti mysql mysql -h 192.168.1.4 -P 3306 -u root -p
```



通过调试，我们发现现在是直接：allow_all_hosts 直接允许访问了

```c++
bool acl_check_host(const char *host, const char *ip)
{
  mysql_mutex_lock(&acl_cache->lock);
  if (allow_all_hosts)
  {
    
    mysql_mutex_unlock(&acl_cache->lock);
    return 0;
  }
  ......
}
```



通过查看代码是在这里设置的，但也看不是太懂，留待日后，今天看的也有点晕了

```c++
static void init_check_host(void)
{
......
  if (acl_users_size && !allow_all_hosts)
  {
      {
        if (my_hash_insert(&acl_check_hosts,(uchar*) acl_user))
        {                                       // End of memory
          // 这里直接设置了
          allow_all_hosts=1;                    // Should never happen
          DBUG_VOID_RETURN;
        }
      }
    }
  }
......
}
```

然后我们就成功访问了



## 总结

今天吧登录认证的代码走了一遍，对MySQL的连接认证有了一个初步的认知，但还是有很多的细节不清楚




使用docker容器连接成功后，我们一直debug下去，发现了tcp连接的监听处理函数：

```cpp
extern "C" void *handle_connection(void *arg)
{
    // 连接准备，今天的host检测和用户认证都是在这个函数里面进行的
    if (thd_prepare_connection(thd))
      handler_manager->inc_aborted_connects();
    else
    {
      while (thd_connection_alive(thd))
      {
	// 看名字就知道是具体命令的处理函数，嘿嘿，找到了下篇文章的函数入口
        if (do_command(thd))
          break;
      }
      end_connection(thd);
    }
    close_connection(thd, 0, false, false);
    ······
}
```



## 参考链接

- [MySQL error code: 1175 during UPDATE in MySQL Workbench](https://stackoverflow.com/questions/11448068/mysql-error-code-1175-during-update-in-mysql-workbench)
- [MySQL篇之初始化数据库后修改root用户密码以及使用指定host登录](https://blog.csdn.net/xu710263124/article/details/119539355)
- [C 库函数 - strchr()](https://www.runoob.com/cprogramming/c-function-strchr.html)