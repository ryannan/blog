* QA

-   Q: vuex requires a Promise polyfill in this browser ?

    A: 浏览器不支持 promise, 安装一个 promise polyfill 即也使用。可支持 IE8+, Chrome, Firefox, IOS 4+, Safari 5+, Opera

    `npm install promise-polyfill --save-exact`

    在对应的入口文件中引入使用
    ```
    import Promise from 'promise-polyfill';

    // To add to window
    if (!window.Promise) {
      window.Promise = Promise;
    }
    ```

    可参考：[https://github.com/taylorhakes/promise-polyfill](https://github.com/taylorhakes/promise-polyfill)
