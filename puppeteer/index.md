# 使用 Puppeteer 构建自动化端到端测试


端到端测试指的是将系统作为一个黑盒，模拟正常用户行为，跨越从前端到后端整个软件系统，是一种全局性的整体测试。

来看本文的示例：

{{< video autoplay="true" src="/videos/puppeteer-eg.mp4" >}}


你在视频中看到的所有操作全部都是由程序自动完成的，就像真实的用户一样，通过这种自动化的方式可以很好的提升我们的测试效率从而保证交付的质量。

完成这样的操作相当简单，只需要 Puppeteer 就够了。Puppeteer 是一个 node 库，通过它提供的高级 API 便可以控制 chromium 或者 chrome ，换句话说，在浏览器中进行的绝大部分人工操作都可以通过在 node 程序中调用 Puppeteer 的 API 来完成。



本文示例中的所有操作无外乎于：

- 获取页面元素
- 键盘输入
- 鼠标操作
- 文件上传
- 执行原生JS


一、打开浏览器跳转页面：

```
const browser = await puppeteer.launch({
headless: false,  // 打开浏览器
defaultViewport: {  // 设置视窗宽高
    width: 1200,
    height: 800
}
});

const page = await browser.newPage();
await page.goto(url);  // 跳转页面
```


二、获取输入框并输入：

```
// -----------------------输入账号密码----------------------------
const input_username = await page.waitForSelector('input[placeholder="用户名"]');
const input_password = await page.waitForSelector('input[placeholder="密码"]');
await input_username.type(username);
await input_password.type(password);
// ---------------------------------------------------
```

通过 page.waitForSelector 方法等待获取到指定的页面元素，也就是 elementHandle , 再直接执行 elementHandle 的 type 方法即可完成键盘输入。


三、通过滑动验证：

1、滑动验证必须要禁用 navigator ，这里通过 page.evaluate 方法直接执行原生JS 即可：

```
await page.evaluate(async () => {  // 滑动验证禁用 navigator
    await Object.defineProperty(navigator, 'webdriver', {get: () => false})
});
```

2、鼠标操作进行验证：

```
async function aliNC (page) {
  const nc = await (await page.waitForSelector('.nc-lang-cnt')).boundingBox();  // 获取滑动验证边界框
  await page.mouse.move(nc.x, nc.y);  // 鼠标移动到起始位置
  await page.mouse.down();  // 鼠标按下
  const steps = Math.floor(Math.random() * 50 + 20);  // 随机 steps
  await page.mouse.move(nc.x + nc.width, nc.y, { steps:  steps});  // 移动到滑块末端位置
  await page.mouse.up();  // 鼠标松开
  await page.waitForTimeout(1200);  // 延时等待验证完成
}
```

先获取到滑动验证的页面元素，再通过 elementHandle 的 boundingBox 方法获取边界框，从而确定 X、Y 二维坐标。

通过 page 的 mouse 相关方法即可进行 move 鼠标移动、down 鼠标按下、up 鼠标松开等操作，需要注意的是我们最好随机生成 steps 来控制鼠标移动的快慢从而避免验证失败。


四、上传文件：

现获取到上传相关的 input 元素即 elementHandle ，然后再调用 elementHandle 的 uploadFile(...filePaths) 方法即可，filePaths 就是文件的路径，如果是相对路径则是相对于当前工作目录。


五、其它：

你会发现几乎所有用户动作就是先获取到相关元素，然后进行键盘或鼠标操作，把它们组合起来就成一整套操作流程。

是自动化的吗？是的，没有人工操作，都是程序在自动进行。

是否真的有效？有效，所有操作都是模拟用户进行的真实行为，从看到前端页面，到提交数据，到请求后端接口，可以说是走了一遍完整的流程，并且整个过程也是可视的，在测试过程中即可发现异常。


最后，我相信 Puppeteer 值得你好好玩一玩，更多用法和 API 还是多翻翻官网，真的很简单。
