# [decrypt-cookies] 实践手记

<!--toc:start-->

- [[decrypt-cookies] 实践手记](#decrypt-cookies-实践手记)
  - [最初的代码架构](#最初的代码架构)
    - [架构的转变](#架构的转变)
    - [转变后带来的复杂性](#转变后带来的复杂性)
  - [为什么采用异步编程](#为什么采用异步编程)
  - [代码库中做的性能优化](#代码库中做的性能优化)
  - [最后](#最后)
  <!--toc:end-->

在开发 [lcode] 时需要进行身份验证来提交题解，最初我让用户手动将 cookies
填写在配置文件中来实现，在后来发现已经有一些程序可以从浏览器的本地 sqlite 数据库中提取出
cookies 的项目，这样用户就不需要在浏览器按下 f12 寻找相关 cookies 了。

我对于密码学和黑客技术知之甚少，只能照着已经实现的项目进行移植，
以及翻阅 [Chromium] 的加密模块的源码。

这里对 rust 代码架构相关工作做一些总结。

## 最初的代码架构

几乎所有基于 [Chromium] 内核的浏览器都可以用同一套解密逻辑将 cookies 进行解密，
剩余的就是细节上的差别，像是还没有将内核更新至最新，以及省去了一些步骤，
不同浏览器之间的最大区别就是他们的文件位置不同。

在最初我将不同浏览器都写入一个枚举中。

```rust
#[non_exhaustive] // 让用户使用时意识到枚举是会增加更多变体的
#[derive(strum::EnumIter)] // 增加迭代支持
pub enum ChromiumBased {
    Chrome,
    Edge,
    Chromium,
}

impl ChromiumBased {
    pub fn cookies_path(&self) -> PathBuf {
        use ChromiumBased::*;
        match self {
            Chrome => "path/to/Chrome/cookies",
            Edge => "path/to/Edge/cookies",
            Chromium => "path/to/Chromium/cookies",
        }.into()
    }
}

pub struct ChromiumGetter {
    conn: DatabaseConnection,
}

impl ChromiumGetter {
    pub fn new(browser: ChromiumBased) -> Self {
        let cookies_path = browser.cookies_path();
        let conn = conn(cookies_path);
        Self { conn }
    }
}
```

看起来很不错，不过每次创建 Getter 时都需要进行判断（也许这里不会对性能造成损耗），
更重要的是枚举将各个浏览器耦合了，意味着每次想要增加新浏览器支持需要将库进行更新，
库作者不可能知道所有的浏览器，用户想要增加新浏览器支持只能使用 cargo 的 patch
增加枚举变体。

### 架构的转变

在后面我了解到了状态机编程模式，通过所有权转移和范型来对结构体上不同范型实现不同的方法
，这里就不展开讲了。

现在代码仓库中的 `ChromiumBuilder<T>` 到 `ChromiumGetter<B>` 之间的转化就用到了这种思想。

我将原始的枚举拆分为独立的结构体，并使用关联常量来标识浏览器的 cookies 位置等信息。

```rust
pub trait ChromiumPath {
    /// Suffix for browser data path
    const BASE: &'static str;
    /// Browser name for [`std::fmt::Display`]
    const NAME: &'static str;
    /// Suffix for cookies data path (sqlite3 database)
    const COOKIES: &str = "Default/Cookies";

    /// Cookies path (sqlite3 database)
    fn cookies(mut base: PathBuf) -> PathBuf {
        push_exact!(base, Self::COOKIES);

        base
    }
}

pub struct Chrome
pub struct Edge
impl ChromiumPath for Chrome {}
impl ChromiumPath for Edge {}

pub struct ChromiumBuilder<T> {
    pub(crate) base: Option<PathBuf>,
    pub(crate) __browser: PhantomData<T>,
}

impl<B: ChromiumPath + Send + Sync> ChromiumBuilder<B> {
    pub const fn new() -> Self {
        Self {
            base: None,
            __browser: PhantomData::<B>,
        }
    }

    pub async fn build(self) -> Result<ChromiumGetter<B>> {
        let base = B::BASE.into();
        let cookies = B::cookies(base.clone());
        // ...
        Ok(ChromiumGetter {
            cookies_query: cookies_query?,
            crypto: crypto?,
            __browser: self.__browser,
        })
    }
}
```

只要是实现了 `ChromiumPath` 的类型都可以作为参数，这样不仅解构了浏览器，
而且用户可以自己添加新的浏览器支持而不需要对库进行修改，这极大的增加了扩展性。

### 转变后带来的复杂性

由于不同浏览器都有了不同的范型参数，相当于不同浏览器都是不同的类型，
以至于无法像之前一样使用迭代器对浏览器迭代，也许在之后需要将
[tidy-browser]
中的宏进行移植来实现类似的功能。

## 为什么采用异步编程

在 node 生态中，有人专门写了一个同步的 [better-sqlite3]，相比于异步版本性能有极大的提升。
如果 [decrypt-cookies] 也采用同步编程是不是可以将性能提升？
在多方面考量后我认为异步不是问题。

- 在 Linux 平台的 [zbus] crate 是异步的，以至于 [secret-service] 也是异步的，
  即使有同步的 API 也只是对异步 API 的包装。
- Cookies 都是在网络请求时才会用到，想必 [reqwest] 的用户不会使用同步 API。
- [sea-orm] 的 cli 工具相当好用，可以很轻松的生成 Entity 省去了很多麻烦。

## 代码库中做的性能优化

### [缓存相关数据][cache linux secret]

在 Linux 平台需要通过 D-Bus 来获取浏览器相关的 secret 值，每次创建一个 `Getter` 都需要从中获取
，其实 crate 是已经内置了一些 secret label，那么就可以将需要的 secret 进行缓存。

### [避免重复的系统调用][avoid unnecessary windows syscall]

Windows 平台需要从系统进程中获取相关 token 才能够调用相关的 Windows 解密 API
，那么只需要[获取一次 pid][accept a pid]，
不需要每次调用都去获取。
这里是一个很简单的逻辑优化，只是做起来麻烦，以及作者是否详细 review 了代码。

### [分支预测优化][separation cookie and login]

```gitsendemail
+/// Maybe use [`std::hint::unlikely`]
+#[cold]
+#[inline(never)]
+fn from_utf8_cold(arg: Vec<u8>) -> std::result::Result<String, std::string::FromUtf8Error> {
+    String::from_utf8(arg)
+}

……

-    pub fn decrypt(&self, ciphertext: &mut [u8]) -> Result<String> {
+    pub fn decrypt(&self, ciphertext: &mut [u8], which: Which) -> Result<String> {
         let (pass, prefix_len) =
             if ciphertext.starts_with(Self::K_APP_BOUND_DATA_PREFIX) && self.pass_v20.is_some() {
                 #[expect(clippy::unwrap_used, reason = "Must be Some, TODO: use let-chains")]
@@ -150,14 +151,19 @@ impl Decrypter {

         cipher
             .decrypt(nonce.into(), raw_ciphertext)
-            .map(|v| {
-                String::from_utf8(v.clone()).unwrap_or_else(|_e| {
-                    #[cfg(feature = "tracing")]
-                    tracing::trace!("Decoding for chromium >=130.x: {_e}");
-                    String::from_utf8_lossy(&v[32..]).into_owned()
-                })
-            })
             .context(error::AesGcmSnafu)
+            .map(|res| match which {
+                Which::Cookie => {
+                    if res.len() > 32 {
+                        String::from_utf8(res[32..].to_vec())
+                    }
+                    else {
+                        crate::from_utf8_cold(res)
+                    }
+                },
+                Which::Login => String::from_utf8(res),
+            })?
+            .context(Utf8Snafu)
     }
 }
```

分支预测就不是从逻辑上进行的优化，而是微架构层面的。
在运行中我通过日志观察到在获取 Cookies 时总是会走到 `unwrap_or_else` 的分支。
进一步研究发现获取登陆信息时总是不需要第二个分支，Cookie 则是不同的浏览器可能内核版本有所区别，但第二个分支是新 chromium 需要的。

于是我将 `Cookie` 和 `Login` 进行了区分，以及使用 `#[cold]`[^1] 表示这个函数是不经常执行的，来得到更好的分支预测性能，不过在粗略的测试中[^2]整体差距只有2%。

### [内存分配优化][unnecessary alloc]

在此 crate 中经常需要获取不同浏览器的路径信息，就需要频繁的对路径进行拼接等操作，
如果简单的使用 `PathBuf::push` 等方法可能会导致频繁的内存分配，这时就需要使用 `PathBuf::with_capacity`, `PathBuf::reserve_exact` 等方法预分配足够的内存，

如果想要对程序进行优化，我想可以先全局搜索一番 `{Vec,PathBuf}::new` 的使用，并考虑是否可以替换为 `with_capacity`。

## 最后

如果对于代码逻辑比较了解的话可以从逻辑上进行很多优化，
在写的代码足够多之后很多地方在写的时候就可以意识到这里会产生什么样的影响，如果不对代码进行 review 和总结
，也不能有很多进步。

[tidy-browser] 中还有一个我比较满意的 crate [binary-cookies] 有时间再聊一聊。

<!-- links -->

[decrypt-cookies]: https://crates.io/crates/decrypt-cookies
[tidy-browser]: https://crates.io/crates/tidy-browser
[binary-cookies]: https://crates.io/crates/binary-cookies
[lcode]: https://github.com/saying121/lcode
[Chromium]: https://source.chromium.org/
[better-sqlite3]: https://github.com/WiseLibs/better-sqlite3
[secret-service]: https://crates.io/crates/secret-service
[zbus]: https://crates.io/crates/zbus
[reqwest]: https://crates.io/crates/reqwest
[sea-orm]: https://crates.io/crates/sea-orm
[cache linux secret]: https://github.com/saying121/tidy-browser/commit/8304474
[accept a pid]: https://github.com/saying121/tidy-browser/commit/f7fef9a
[avoid unnecessary windows syscall]: https://github.com/saying121/tidy-browser/commit/edbf3fd
[separation cookie and login]: https://github.com/saying121/tidy-browser/commit/b08dfc2
[unnecessary alloc]: https://github.com/saying121/tidy-browser/commit/2f97f01

[^2]: 像是从系统获取 secret 的时间，访问 sqlite 的时间并未排除，我只想要看到它是否生效

[^1]: 似乎 `#[cold]` 已经起到了 `#[inline(never)]` 的作用，也许 `#[inline(never)]` 是不必要的，希望 `std::hint::unlikely` 早日稳定
