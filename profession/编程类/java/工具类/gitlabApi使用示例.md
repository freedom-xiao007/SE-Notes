# Gitlab Java API 使用示例
***

一起养成写作习惯！这是我参与「掘金日新计划 · 4 月更文挑战」的第17天，[点击查看活动详情](https://juejin.cn/post/7080800226365145118)。

## 简介
在开发中，偶尔会有一些关于Gitlab的二开需求，本文将介绍如果在Java中使用Gitlab提供的API

## 功能介绍
示例中代码，主要的功能如下：

- 读取整个仓库中的所有文件，读取后进行相关的处理
- 使用Webhook，接收gitlab的Webhook请求，进行代码push事件的监听处理

下面具体的示例代码

## 代码示例
### 依赖导入
在maven中导入gitlab api的仓库

```maven
  <dependency>
            <groupId>org.gitlab4j</groupId>
            <artifactId>gitlab4j-api</artifactId>
            <version>4.19.0</version>
        </dependency>
```

### 配置相关参考
示例中使用的Spring boot，我们在配置文件中添加相关的gitlab配置信息

主要是服务地址和相关的认证信息等，如下

```yml
application:
  gitlab:
    # gitlab访问地址
   host: http://127.0.0.1:81/
    # gitlab 访问的token
    accessToken: xxxxxxxxxxx
    # 对应用户名
    namespace: user
    # 工程名称
    projectName: testProject
    # 工程对应的分支,用于区分环境
    branch: test
```

下面是对应的配置类：

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "application.gitlab")
public class GitlabConfig {

    private String host;
    private String accessToken;
    private String namespace;
    private String projectName;
    private String branch;
}
```

### webhook回调接口
这里简单介绍下Webhook，其主要作用是：

- 当设置的仓库事件发生时（比如代码push，进行了一次提交）
- gitlab就会调用这个回调的接口，发送这次push相关的信息给你的服务
- 回调时里面有相关的提交信息，你可以根据相关的信息来处理

下面这个是回调的接口:

```java
@RestController
@RequestMapping("/")
public class GitlabController {

    // gitlab仓库push操作时的回调接口
    @PostMapping("/event/push")
    public RespResult<String> pushEvent(@RequestBody GitlabPushEvent gitlabPushEvent) {
        gitlabService.pushEvent(gitlabPushEvent);
        return RespResult.<String>builder().code("200").data("success").build();
    }
}
```

这个是回调的参考，里面有很多，这个类只取了主要的信息，如下所示：

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class GitlabPushEvent {

    // 这个提交的分支信息
    private String ref;
    // 提交的详细信息
    private List<Commit> commits;

    @Data
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public static class Commit {

        // 新增的文件列表
        private List<String> added;
        // 修改的文件列表
        private List<String> modified;
        // 删除的文件列表
        private List<String> removed;
    }
}
```

### 相关的处理逻辑
下面是简单使用示例

```java
@Slf4j
@Service
public class GitlabService {

    private final GitlabConfig gitlabConfig;
    private final GitLabApi gitLabApi;

    public GitlabService(final GitlabConfig gitlabConfig, final GitbookConfig gitbookConfig) {
        this.gitlabConfig = gitlabConfig;
	// 传入gitlab服务地址和token，初始化服务
        gitLabApi = new GitLabApi(gitlabConfig.getHost(), gitlabConfig.getAccessToken());
    }

    private void readAllFile(final Project project) {
	// 读取指定项目的，指定分支的工程
	final Project project = gitLabApi.getProjectApi().getProject(gitlabConfig.getNamespace(), gitlabConfig.getProjectName());
	// 获取工程的所有文件
        final List<String> allFil = getAllFiles(project.getId(), gitlabConfig.getDirectory(), gitlabConfig.getBranch());
	// 遍历获取所有文件，进行相关的处理
        allFil.forEach(filePath -> {
	    // 读取文件内容
            final InputStream inputStream = gitLabApi.getRepositoryFileApi().getRawFile(projectId, branch, filePath);
            final BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
            final StringBuilder stringBuilder = new StringBuilder();
            while (reader.ready()) {
                final String lineContent = reader.readLine();
            }
            stringBuilder.append(lineContent);
	    // todo 进行相关的处理
            }
        });
    }

   
   // 读取工程仓库的所有文件列表
    @SneakyThrows
    private List<String> getAllFiles(final Integer projectId, final String directory, final String branch) {
        List<String> fileNames = new ArrayList<>();
        gitLabApi.getRepositoryApi().getTree(projectId, directory, branch).forEach(file -> {
		// 如果当前是目录，则继续获取其下的文件列表
            if (file.getType().equals(TreeItem.Type.TREE)) {
                fileNames.addAll(getAllFiles(projectId, file.getPath(), branch));
                return;
            }
	    // 如果是文件类型，直接添加
            final String filePath = String.join("/",directory, file.getName());
            fileNames.add(filePath);
            log.info("add file: {}", filePath);
        });
        return fileNames;
    }

    // Webhook回调函数处理，根据自己的需求进行处理
    @SneakyThrows
    public void pushEvent(GitlabPushEvent gitlabPushEvent) {
        log.info("收到Gitlab Webhook请求");
        if (!gitlabPushEvent.getRef().equals(String.format("refs/heads/%s", gitlabConfig.getBranch()))) {
            log.info("非设定分支的push事件，不进行更新");
            return;
        }

        for (GitlabPushEvent.Commit commit: gitlabPushEvent.getCommits()) {
	    for (String filePath: commit.getAdded()) {
		    // todo 处理新增的文件
	    }
	    for (String filePath: commit.getModified()) {
		    // todo 处理修改的文件
	    }
	    for (String filePath: commit.getRemoved()) {
		    // todo 处理删除的文件
	    }
        }
    }
}
```