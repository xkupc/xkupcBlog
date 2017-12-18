---
title: 简单爬虫（二）-selenium的使用
date: 2017-11-18 17:10:39
tags: [java,spring-boot]
categories: springboot
---
### 前言
上一篇我已经解除了爬虫的一些理念，那就是模拟http请求网页，后去返回的静态html，通过jsoup解析，获取页面的上的你想要的数据。但是有些情况下，页面数据是动态通过js加载的，这时候你从返回的静态页里是获取不到任何有效数据的，还有一种情况是有些网页下只有登录了，才能看到你想要的数据。这个时候selenium的作用就体现出来了。
### selenium
selenium是一款自动化测试工具，通过它我们可以通过代码的来模拟操控浏览器鼠标来获取我们想要的数据。今天我们来模拟一下自动登录腾讯视频。使用selenium我们需要引入依赖：
<!--more-->
```js
 <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>3.7.1</version>
            <exclusions>
                <exclusion>
                    <groupId>com.google.guava</groupId>
                    <artifactId>guava</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>22.0</version>
        </dependency>
```
这里是有个坑的，selenium3.7.1的版本依赖的guava版本太高，有个方法已经找不到，会导致报错，所以我单独引入了低版本的guava。
要想selenium去操控浏览器，就需要一个驱动去做适配。小编用的chrome，所以自然选择chromedriver。
下载地址：http://chromedriver.storage.googleapis.com/index.html
注意这里也是有坑的，chromedriver一定要和你使用的浏览器版本有对应关系，小编在这里呗坑了好久。这里我给出一个网上的大神出的映射表：http://blog.csdn.net/huilan_same/article/details/51896672
第三个坑就是启动selenium的时候，由于其socket端口可能超过window默认允许的最大端口号大小（win7可能会有），需要在系统配置里设置一下最大端口号。具体操作的说明地址，小编忘记了。。。。
### 代码
```java
public void login() {
        WebDriver driver;

        System.setProperty("webdriver.chrome.driver", "D:/tools/chromedriver/chromedriver.exe");//这一步必不可少

        driver = new ChromeDriver();

        driver.manage().timeouts().implicitlyWait(2, TimeUnit.SECONDS);

        driver.get("https://v.qq.com/");
        try {
            WebElement webElement = driver.findElement(By.xpath("//div[@class='quick_item quick_user']"));
            Actions action = new Actions(driver);
            //模拟鼠标停到头像上
            action.moveToElement(webElement).perform();
            //必须休眠一下，等登录框按钮出现
            Thread.sleep(500L);
            webElement = driver.findElement(By.xpath("//div[@class='quick_pop_btn']"));
            webElement.click();
            webElement = driver.findElement(By.xpath("//div[@class='login_btns']"));
            WebElement qqloginElement = webElement.findElement(By.xpath("//a[@class='btn_qq _login_type_item']"));
            qqloginElement.click();
            Thread.sleep(500L);
            //切换iframe,使用账号密码登陆
            driver.switchTo().frame("_login_frame_quick_");
            WebElement loginElement = driver.findElement(By.xpath("//div[@class='login']"));
            WebElement loginElementWithAccount = loginElement.findElement(By.xpath("//div[@class='bottom hide']"));
            loginElementWithAccount = loginElementWithAccount.findElement(By.xpath("//a[@id='switcher_plogin']"));
            loginElementWithAccount.click();
            loginElementWithAccount = driver.findElement(By.xpath("//div[@class='web_qr_login']"));
            loginElementWithAccount = loginElementWithAccount.findElement(By.xpath("//div[@class='web_qr_login_show']")).findElement(By.xpath("//div[@class='web_login']")).
                    findElement(By.xpath("//div[@class='login_form']"));
            loginElementWithAccount.findElement(By.xpath("//div[@class='uinArea']")).findElement(By.id("u")).clear();
            //设置账号
            loginElementWithAccount.findElement(By.xpath("//div[@class='uinArea']")).findElement(By.id("u")).sendKeys("XXXX");
            loginElementWithAccount.findElement(By.xpath("//div[@class='pwdArea']")).findElement(By.id("p")).clear();
            //设置密码
            loginElementWithAccount.findElement(By.xpath("//div[@class='pwdArea']")).findElement(By.id("p")).sendKeys("XXXXX");
            loginElementWithAccount.findElement(By.xpath("//div[@class='submit']")).findElement(By.xpath("//a[@class='login_button']")).click();
            Thread.sleep(100L);
            Set<Cookie> cookies = driver.manage().getCookies();
            //保存cookie
            redisTemplate.opsForSet().add("cookie", cookies);
        } catch (Exception e) {
            System.err.println(e);
        } finally {
            if (null != driver) {
                driver.close();
            }
        }
```