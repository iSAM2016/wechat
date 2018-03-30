### JS-SDK
>


1. 签名

    看到网上的大部分问题都集中在签名部分，请大家一定请熟读[微信JS-SDK说明文档](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421141115)`附录5-常见错误及解决方法` 部分。都能解决问题


**注意**

        *. 在计算签名的过程中，如果url总是不对请 实验 `首页的url`或 window.location.href。因此真实的请求路径与拿去生成签名的路径是一致的，千万记住这条

        *. 前端需要用js获取当前页面除去'#'hash部分的链接（可用location.href.split('#')[0]获取,而且需要encodeURIComponent

        *. vue每次刷新页面，都需要重新配置SDK，**使用JS_SDK必须先注入配置信息**

        *. [关于html5-History模式在微信浏览器内的问题 #481](https://github.com/vuejs/vue-router/issues/481)

        *. IOS：微信IOS版，微信安卓版，每次切换路由，SPA的url是不会变的，发起签名请求的url参数必须是当前页面的url就是最初进入页面时的url(entryUrl.js)

        *. Android：微信安卓版，每次切换路由，SPA的url是会变的，发起签名请求的url参数必须是当前页面的url(不是最初进入页面时的)(entryUrl.js)

        *. 登录微信公众平台进入“公众号设置”的“功能设置”里填写“JS接口安全域名”。

        *. 域名格式：如果你的项目域名是http://test.domain.com,那么JS接口安全域名为test.domain.com

        *. timestamp: , // 生成签名的时间戳，精确到秒 秒  秒  秒  

        *. nonceStr: '', // 必填，生成签名的随机串  


```
// entryUrl.js
 全局存储进入SPA的url（window.entryUrl），Android不变，依旧是获取当前页面的url，IOS就使用window.entryUrl
// 记录进入app的url，后面微信sdk
if (window.entryUrl === '') {
  window.entryUrl = location.href.split('#')[0]
}
// 进行签名的时候
url: isAndroid() ? location.href.split('#')[0] : window.entryUrl
```


2. 签名 （后台）

   后台生成签名后返回给前台使用，很多微信api需要这个签名。

   生成后可以验证一下签名是否正确    [签名校验工具](http://work.weixin.qq.com/api/jsapisign)

   ```php
   <?php
   namespace Vendor\wxpay;//命名空间
   /**
    * Class wxpay
    * @package Vendor\wxpay
    * @name 用于签名生产
    * @author weikai
    */
   class wxpay {
       private $appId;
       private $appSecret;

       public function __construct($appId, $appSecret) {
           $this->appId = $appId;
           $this->appSecret = $appSecret;
       }
       //获取签名
       public function getSignPackage() {
           $jsapiTicket = $this->getJsApiTicket();//获取JsApiTicket

           // vue管理路由 url参数建议前台传递
           $protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off' || $_SERVER['SERVER_PORT'] == 443) ? "https://" : "http://";
           // $url = "$protocol$_SERVER[HTTP_HOST]$_SERVER[REQUEST_URI]";//如果后台获取url
           $url = I('get.frontUrl');//获取前台传递的url
           $timestamp = time();//现在时间戳
           $nonceStr = $this->createNonceStr();//生成随机字符串

           // 这里参数的顺序要按照 key 值 ASCII 码升序排序
           $string = "jsapi_ticket=$jsapiTicket&noncestr=$nonceStr&timestamp=$timestamp&url=$url";

           $signature = sha1($string);//sha1加密排序后的参数生产签名
   		//将所有参数赋值到数组
           $signPackage = array(
               "appId"     => $this->appId,
               "nonceStr"  => $nonceStr,
               "timestamp" => $timestamp,
               "url"       => $url,
               "signature" => $signature,
               "rawString" => $string
           );

           return $signPackage;
       }
     
   	//生成随机字符串
       private function createNonceStr($length = 16) {
           $chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
           $str = "";
           for ($i = 0; $i < $length; $i++) {
               $str .= substr($chars, mt_rand(0, strlen($chars) - 1), 1);
           }
           return $str;
       }
   	//生成JsApiTicket
       private function getJsApiTicket() {
           // jsapi_ticket 全局存储与更新
           $data = json_decode(S('jsapi_ticket'));
         //如果缓存中的JsApiTicket 不在有效期内重新生成JsApiTicket 
           if ($data->expire_time < time()) {
               $accessToken = $this->getAccessToken();//获取AccessToken
               // 如果是企业号用以下 URL 获取 ticket
               // $url = "https://qyapi.weixin.qq.com/cgi-bin/get_jsapi_ticket?access_token=$accessToken";
               $url = "https://api.weixin.qq.com/cgi-bin/ticket/getticket?type=jsapi&access_token=$accessToken";
               $res = json_decode($this->httpGet($url));
               $ticket = $res->ticket;
             //更新有效期存入缓存
               if ($ticket) {
                   $data->expire_time = time() + 7000;
                   $data->jsapi_ticket = $ticket;
                   S('jsapi_ticket',json_encode($data));
               }
           } else {
             //否则缓存中的JsApiTicket 在有效期内就直接用
               $ticket = $data->jsapi_ticket;
           }

           return $ticket;
       }
   	//获取全局AccessToken
       public function getAccessToken() {
           // access_token 全局存储与更新
           $data = json_decode(S('access_token'));
           if ($data->expire_time < time()) {
               // 如果是企业号用以下URL获取access_token
               // $url = "https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=$this->appId&corpsecret=$this->appSecret";
               $url = "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=$this->appId&secret=$this->appSecret";
               $res = json_decode($this->httpGet($url));
               $access_token = $res->access_token;
               if ($access_token) {
                   $data->expire_time = time() + 7000;
                   $data->access_token = $access_token;
                   S('access_token',json_encode($data));
               }
           } else {
               $access_token = $data->access_token;
           }
           return $access_token;
       }

       //curl 请求
       private function httpGet($url) {
           $curl = curl_init();
           curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
           curl_setopt($curl, CURLOPT_TIMEOUT, 500);
           // 为保证第三方服务器与微信服务器之间数据传输的安全性，所有微信接口采用https方式调用，必须使用下面2行代码打开ssl安全校验。
           // 如果在部署过程中代码在此处验证失败，请到 http://curl.haxx.se/ca/cacert.pem 下载新的证书判别文件。
           curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, true);
           curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, true);
           curl_setopt($curl, CURLOPT_URL, $url);

           $res = curl_exec($curl);
           curl_close($curl);

           return $res;
       }


   }
   ```

   ​


3. 签名（前台）

    * 微信配置
    * 微信扫一扫
    * 分享到朋友圈
    * 朋友的设置
    * 分享的基本配置
    * 微信支付

```
/**
 * Created by  isam2016 
 */

import wx from 'weixin-js-sdk';
import axios from 'axios';

var JsWeChatApis = [
  'checkJsApi',
   请补全列表
];

var isWeChatReady = false;
var sharePage = "default";
 
export default class AlWeChat {
  constructor(object) {
    this.object = object;
    this.wxConfig();// 初始微信配置
  }
  isWeChatReady() {
    return isWeChatReady;
  }
  /**
   * 微信配置
   * @Author   Hybrid
   * @DateTime 2017-11-21
   * @param    {}   router [单页面应用，由前台通知URL，此方法并不完美，探寻更好的方法]
   * @return   {[type]}          [description]
   */
  wxConfig() {
    let self = this;
    axios.get('/home/OrderConfirm/wxConfig', {
      params: {
        frontUrl: location.href.split('#')[0]// 前台吧url 传到后台 而且需要encodeURIComponent
      }
    }).then(function(response) {
      var attachment = response.data.data;
      // console.log(attachment);
      wx.config({
        debug: false,
        appId: attachment.appId,
        timestamp: attachment.timestamp, // 支付签名时间戳小写s 时间戳(timestamp)值要记住精确到秒，不是毫秒。
        nonceStr: attachment.nonceStr,//支付签名随机串，不长于 32 位,大写s
        signature: attachment.signature,
        url: attachment.url,
        jsApiList: JsWeChatApis
      });
      wx.ready(function () {
        isWeChatReady = true;
        self.object && self.wxQDetailShare()
      });
      wx.error(function (res) {
        //console.log(JSON.stringify(res));
      });

    }).catch(function (error) {
      // console.log(error)
    });
  }

  /**
   * 微信扫一扫
   * @Author   Hybrid
   * @DateTime 2017-11-21
   * @return   {[type]}   [description]
   */
  wxScanQRCode(fn) {
    wx.scanQRCode({
      needResult: 1, // 默认为0，扫描结果由微信处理，1则直接返回扫描结果，
      scanType: ["qrCode"], // 可以指定扫二维码还是一维码，默认二者都有
      success: function (res) {
        var result = res.resultStr; // 当needResult 为 1 时，扫码返回的结果
        fn(result);
      }
    });
  }

  /**
   * 分享到朋友圈+朋友的设置
   * 可以动态设置
   * @Author   Hybrid
   * @DateTime 2017-11-21
   * @param    {}   data [展示数据]
   * @param    {[type]}   eqid [description]
   * @return   {[type]}        [description]
   */
  wxQDetailShare() {
    var config = {
      title: 'XXX',
      desc: 'XXX',
      imgUrl: 'XXX',
      link: 'XXX',
    };
    var shareConfig = {
      message: config,
      timeLine: {
        title: config.title,
        desc: config.desc,
        imgUrl: config.imgUrl,
        link: config.link,
      },
    };
    this.wxShare(shareConfig);
  }

  /**
   * 分享的基本配置
   * @Author   Hybrid
   * @DateTime 2017-11-21
   * @param    {}   shareConfig [不同类型的分享有不同的配置]
   * @return   {[type]}               [description]
   */
  wxShare(shareConfig) {
    let self = this;
    if (isWeChatReady) {
      /**
       * 分享到朋友圈
       * @Author   Hybrid
       */
      wx.onMenuShareTimeline({
        title: shareConfig.timeLine.title, // 分享标题
        link: shareConfig.timeLine.link, // 分享链接
        imgUrl: shareConfig.timeLine.imgUrl, // 分享图标
        success: function () {
          self.object.closecovershow();
          // 用户确认分享后执行的回调函数
        },
        cancel: function () {
          self.object.closecovershow();
          // 用户取消分享后执行的回调函数
        }
      });

      /**
       * 分享给朋友
       * @Author   Hybrid
       */
      wx.onMenuShareAppMessage({
        title: shareConfig.message.title, // 分享标题
        desc: shareConfig.message.desc, // 分享描述
        link: shareConfig.message.link, // 分享链接
        imgUrl: shareConfig.message.imgUrl, // 分享图标  type: '', // 分享类型,music、video或link，不填默认为link
        type: '', // 分享类型,music、video或link，不填默认为link
        dataUrl: '', // 如果type是music或video，则要提供数据链接，默认为空
        success: function () {
          self.object.closecovershow();
          // 用户确认分享后执行的回调函数
        },
        cancel: function () {
          self.object.closecovershow();
          // 用户取消分享后执行的回调函数
        }
      });
    }
  }


  /**
   * 微信支付
   * @Author   Hybrid
   * @DateTime 2017-11-21
   * @param    {string}   router 单页面应用，由前台通知URL
   * @return   {[type]}          [description]
   */
  payWeChat(order_num) {
    let self = this;
    axios.get('/home/OrderConfirm/orderPay', {
      params: {
        type: 'weixin',
        frontUrl: location.href.split('#')[0],
        order_num
      }
    }).then(function (response) {
      var attachment = response.data.data;
      localStorage.setItem(wechatCode, '');
      //alert(location.href)
      WeixinJSBridge.invoke('getBrandWCPayRequest', {
        "appId": attachment.appId,
        "timeStamp": attachment.timeStamp,
        "nonceStr": attachment.nonceStr,
        "package": attachment.package,
        "signType": 'MD5',
        "paySign": attachment.paySign,
      }, function (res) {
        // console.log(res);
        if (res.err_msg == "get_brand_wcpay_request:ok") {
          self.object.$router.push("/paysuccess");
        } else if (res.err_msg == "get_brand_wcpay_request:cancel") {
          self.object.$router.push("/ordercenter");
        } else {
          // localStorage.setItem(wechatCodeOld, '');
          localStorage.setItem(wechatCode, '');
          //alert("支付失败!" + JSON.stringify(res) + "当前路径" + location.href);
          // alert("支付失败!" + JSON.stringify(res));
          //   resolve(-1);
        }
      })

    }).catch(function (err) {
      console.log(JSON.stringify(err));
    })
  }
}
```

调用 new AlWeChat(this)  //this = vue 


tips
在调用微信支付前，需要从后端发起 unifiedorder 统一下单请求获取到 prepay_id，通过 prepay_id、AppID、商品平台的 key 等参数生成签名，最后调用 wx.chooseWXPay 发起微信支付。
注意，生成支付签名这边有很多的坑，比如 timestamp 在 wx.chooseWxPay 里是小写，而在生成签名时使用的是 timeStamp。尽量使用成熟的库来避免这种不知所谓的坑。
统一下单请求时会要求传入参数 notify_url，这个是异步接受微信支付结果通知的回调地址。注意该地址不能携带参数。举例来说 https://www.xxx.com?a=1，回调时 a=1会被省略。

