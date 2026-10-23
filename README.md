# 警告
**请勿在酷安发表任何关于恢复旧版评论区的直接教程、间接暗示、技术原理（HTTP改写）等内容，否则你会收到：**
![img](fkcoolapk.png)

# CoolapkReviver
让旧版酷安通过服务端版本校验，修复酷安评论区接口异常

> 测试基准：本项目基于酷安 **13.0.1** 版本开发与实测，该版本**未使用 360 加固**，`okhttp3` 类以明文 dex 存在，可直接 hook。

## 解决什么问题
酷安服务端对 v6 接口做了版本白名单。客户端版本过低时，请求统一返回 403，App 弹出
「评论接口异常，请尝试升级到新版酷安」，评论区、私信直接不可用。
官方解法是升级 App。本模块的解法是让旧版客户端在请求时自称新版，通过服务端的版本校验，
从而继续使用原有功能，不需要升级、不需要换包。
模块只改 HTTP 请求头里的版本标识，不动 APK、不改界面、不删广告、不加权限，
对 App 的其他行为没有影响。

## 原理
旧版酷安的所有版本标识由 `com.coolapk.market.AppConfig` 生成，再由
`HttpClientFactory$CoolMarketHeaderInterceptor` 逐对写入 OkHttp 请求：
```java
List<String> headers = appConfig.createHeaders();
for (int i = 0; i < headers.size(); i += 2) {
    builder.header(headers.get(i), headers.get(i + 1));
}
```
服务端靠这些头判断客户端版本。模块 hook `okhttp3.Request$Builder#header` 与 `#addHeader`，
在写入前替换版本相关的头，让服务端看到一个符合白名单的版本号。
因为 hook 的是 OkHttp 而不是酷安的混淆类，客户端小版本升级一般不需要改模块。

> 模块可行性的前提是目标类可被直接加载：当前测试版本 13.0.1 未使用 360 加固，
> dex 未加密，`Class.forName("okhttp3.Request$Builder")` 可正常解析；
> 若目标版本接入加固壳，hook 入口将不再可用（详见「已知限制」）。

## 改写内容
| Header | 原值 | 改写为 |
| --- | --- | --- |
| `X-App-Version` | `13.0.1` | `16.6.4` |
| `X-App-Code` | `2301171` | `2609291` |
| `X-Api-Version` | `13` | `16` |
| `X-App-Supported` | `2301171` | `2310232` |
| `User-Agent` | `+CoolMarket/13.0.1-2301171-...` | `+CoolMarket/16.6.4-2609291-...` |
未命中以上名称的请求原样放行。

## 安装
1. 从 Release 下载 `CoolapkReviver-1.0.apk` 并安装。
2. LSPosed 管理器 → 模块 → 启用 CoolapkReviver。
3. 作用域勾选酷安（`com.coolapk.market`）。
4. 强行停止酷安后重新打开。

> 已在酷安 13.0.1（未加固版）上实测通过；其余版本请自行验证后使用。

## 日志
只在主进程启动时输出一行，子进程与请求过程不输出。
```
I CoolapkReviver: hook ok: 16.6.4 / 2609291 / api 16
```
hook 失败时：
```
E CoolapkReviver: hook failed: java.lang.ClassNotFoundException: okhttp3.Request$Builder
```

## 构建
环境：JDK 17、Android SDK Platform 35、Build-Tools 35.0.0。
```bash
./gradlew assembleRelease
```
产物在 `app/build/outputs/apk/release/app-release.apk`。
`app/libs/libxposed-api-102.jar` 是 `io.github.libxposed:api:102.0.0` 的 classes.jar，
以本地文件方式引入，保证离线可构建。若想改用仓库依赖，把它删掉并改成：
```kotlin
compileOnly("io.github.libxposed:api:102.0.0")
```

## 目录
```
app/src/main/java/io/github/fentia/coolapkreviver/ModuleMain.java   模块逻辑
app/src/main/resources/META-INF/xposed/module.prop                    模块声明
app/src/main/resources/META-INF/xposed/java_init.list                 入口类
app/src/main/resources/META-INF/xposed/scope.list                     作用域
```

## 调整目标版本
改 `ModuleMain.java` 顶部四个常量后重新构建：
```java
private static final String VERSION_NAME = "16.6.4";
private static final String VERSION_CODE = "2609291";
private static final String API_VERSION = "16";
private static final String APP_SUPPORTED = "2310232";
```
`API_VERSION` 取 `VERSION_NAME` 的首段，两者需要一起改。服务端提高版本门槛后，
更新这组常量重新打包即可，模块不会联网获取版本。

## 已知限制
- 只处理 OkHttp 出站请求头。评论、私信、点赞等 v6 接口都走这条链路；若有走 WebSocket 的通道则不在覆盖范围内。
- 当前测试基准 13.0.1 **未使用 360 加固**，这是 hook 能生效的前提。若日后使用的酷安版本接入 360 加固，
  dex 会被加密、由壳在运行时动态加载，`Class.forName("okhttp3.Request$Builder")` 将失败并在日志中报错，
  需改为 hook `AppConfig` 的混淆方法，或在加固壳完成类加载后再 hook。
- 若酷安日后混淆 `okhttp3`（未加固的前提下），`Class.forName` 同样会失败，处理方式同上。
- 改写的是请求头里的版本号，客户端代码仍是旧版。若服务端将来在 v6 接口上新增字段，旧版解析可能不完整。

## 免责声明
仅供学习与研究使用。使用本模块产生的任何后果由使用者自行承担。
