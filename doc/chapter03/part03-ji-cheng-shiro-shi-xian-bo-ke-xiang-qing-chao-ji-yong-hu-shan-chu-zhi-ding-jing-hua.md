# 3. 集成 Shiro 实现博客详情-超级用户、删除、置顶、精华

```text
blog
├─src
│  └─main
│      ├─java
│      │  └─org
│      │      └─myslayers
│      │          ├─exception
│      │          │      GlobalException.java
│      │          │
│      │          ├─controller
│      │          │      BaseController.java
│      │          │      AdminController.java
│      │          │
│      │          ├─shiro
│      │          │      AccountRealm.java    
│      │
│      └─resources
│          ├─templates
│          │  │  error.ftl
│          │  │
│          │  └─post
│          │         detail.ftl
```

## 3.1 博客详情-超级用户

* `AccountRealm.java` ：过滤器，授权 id=1 的用户 admin 为 超级用户

  ```java
  /**
  * Shiro过滤器：授权 / 认证
  */
  @Component
  public class AccountRealm extends AuthorizingRealm {

    @Autowired
    UserService userService;

    /**
     * doGetAuthorizationInfo（授权）：
     * <p>
     * 需要判断是否有访问某个资源的权限
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        AccountProfile profile = (AccountProfile) principals.getPrimaryPrincipal();
        // 给id=1的admin用户，赋予admin角色
        if(profile.getId() == 1) {
            SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
            info.addRole("admin");
            return info;
        }
        return null;
    }
  }
  ```

## 3.2 博客详情-删除、置顶、精华

* `detail.ftl` ：模板引擎，使用&lt;@shiro.hasRole name="admin"&gt;[/@shiro](mailto:/@shiro)标签对【删除】、【置顶】、【加精】进行处理，因此，该功能只能【登录 admin 超级用户】

  \`\`\`injectedfreemarker

  &lt;\#--1.1.2 文章标签--&gt;

  ${post.categoryName}

&lt;\#if post.level gt 0&gt;置顶&lt;/\#if&gt; &lt;\#if post.recommend&gt;精帖&lt;/\#if&gt;

 删除 删除置顶取消置顶加精取消加精

 [${post.commentCount}](part03-ji-cheng-shiro-shi-xian-bo-ke-xiang-qing-chao-ji-yong-hu-shan-chu-zhi-ding-jing-hua.md#comment) ${post.viewCount} &lt;/div&gt;

```text
### 3.3 博客详情-数据接口
- `AdminController.java` ：控制层，根据前端传来的 3 个参数：id、rank、field，对功能进行实现
```java
@Controller
public class AdminController extends BaseController {

    /**
     * 访问 /post/{id} 的文章时，如果为 admin 超级管理员，则可以管理该文章，例如【删除】、【置顶】、【加精】
     *
     * 实现思路：
     * 1.AccountRealm.java 中的 doGetAuthorizationInfo() 方法 -> 授权 id = 1 的用户 admin 为 超级管理员
     * 2.detail.ftl 页面，使用<@shiro.hasRole name="admin"></@shiro>标签对【删除】、【置顶】、【加精】进行处理，因此，该功能只能【登录admin超级管理员账户】
     * 3.根据前端传来的 3 个参数：id、rank、field，对功能进行实现
     *
     * @param id    post.id
     * @param rank  0表示取消（取消置顶、取消加精），1表示操作（删除、置顶、加精）
     * @param field 操作类型：删除（field：delete）、置顶（field：stick）、加精（field：status）
     */
    @ResponseBody
    @PostMapping("/admin/jie-set")
    public Result jetSet(Long id, Integer rank, String field) {
        //根据id判断该文章是否被删除
        Post post = postService.getById(id);
        Assert.notNull(post, "该文章已被删除");

        //删除
        if ("delete".equals(field)) {
            postService.removeById(id);
            return Result.success();
        } else if ("status".equals(field)) {
            //置顶
            post.setRecommend(rank == 1);
        } else if ("stick".equals(field)) {
            //加精
            post.setLevel(rank);
        }

        postService.updateById(post);
        return Result.success();
    }
}
```

## 3.4 其他-全局异常

* `GlobalException.java` ：全局异常，分别对 Ajax 异常请求、Web 异常请求进行处理

  ```java
  /**
  * 全局异常
  */
  @Slf4j
  @ControllerAdvice
  public class GlobalException {

    @ExceptionHandler(value = Exception.class)
    public ModelAndView handler(HttpServletRequest req, HttpServletResponse resp, Exception e) throws IOException {
        //Ajax异常请求
        String header = req.getHeader("X-Requested-With");
        if (header != null && "XMLHttpRequest".equals(header)) {
            resp.setContentType("application/json;charset=UTF-8");
            resp.getWriter().print(JSONUtil.toJsonStr(Result.fail(e.getMessage())));
            return null;
        }
        //Web异常请求
        ModelAndView modelAndView = new ModelAndView("error");
        modelAndView.addObject("message", e.getMessage());
        return modelAndView;
    }
  }
  ```

* `error.ftl` ：模板引擎，将 message 错误信息进行显示

  \`\`\`injectedfreemarker

  &lt;\#--宏layout.ftl（导航栏 + 页脚）--&gt;

  &lt;\#include "/inc/layout.ftl" /&gt;

&lt;\#--【三、填充（导航栏 + 页脚）】--&gt; &lt;@layout "错误页面"&gt;

&lt;\#--【二、分类】--&gt; &lt;\#include "/inc/header-panel.ftl" /&gt;

${message}

  
    layui.cache.page = '';  
  

[/@layout](mailto:/@layout) \`\`\`
