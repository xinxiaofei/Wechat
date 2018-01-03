# Wechat
微信JSSDk验签


```javascript
'use strict' //设置为严格模式

const https = require('https'); //引入 htts 模块
const util = require('util'); //引入 util 工具包
const fs = require('fs'); //引入 fs 模块
const urltil = require('url');//引入 url 模块
const jsSHA = require('jssha');
var crypto = require('crypto');
const accessTokenJson = require('./access_token.json'); //引入本地存储的 access_token
const jsapiTicketJson = require('./jsapi_ticket.json'); //引入本地存储的 jsapi_ticket


/**
 * 构建 WeChat 对象 即 js中 函数就是对象
 * @param {JSON} config 微信配置文件 
 */
var WeChat = function(config){
    //设置 WeChat 对象属性 config
    this.config = config;
    //设置 WeChat 对象属性 token
    this.token = config.token;
    //设置 WeChat 对象属性 appID
    this.appID = config.appID;
    //设置 WeChat 对象属性 appScrect
    this.appScrect = config.appScrect;
    //设置 WeChat 对象属性 apiDomain
    this.apiDomain = config.apiDomain;
    //设置 WeChat 对象属性 apiURL
    this.apiURL = config.apiURL;

    /**
     * 用于处理 https Get请求方法
     * @param {String} url 请求地址 
     */
    this.requestGet = function(url){
        return new Promise(function(resolve,reject){
            https.get(url,function(res){
                var buffer = [],result = "";
                //监听 data 事件
                res.on('data',function(data){
                    buffer.push(data);
                });
                //监听 数据传输完成事件
                res.on('end',function(){
                    result = Buffer.concat(buffer).toString('utf-8');
                    //将最后结果返回
                    resolve(result);
                });
            }).on('error',function(err){
                reject(err);
            });
        });
    }

    /**
     * 用于处理 https Post请求方法
     * @param {String} url  请求地址
     * @param {JSON} data 提交的数据
     */
    this.requestPost = function(url,data){
        return new Promise(function(resolve,reject){
            //解析 url 地址
            var urlData = urltil.parse(url);
            //设置 https.request  options 传入的参数对象
            var options={
                //目标主机地址
                hostname: urlData.hostname, 
                //目标地址 
                path: urlData.path,
                //请求方法
                method: 'POST',
                //头部协议
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded',
                    'Content-Length': Buffer.byteLength(data,'utf-8')
                }
            };
            var req = https.request(options,function(res){
                var buffer = [],result = '';
                //用于监听 data 事件 接收数据
                res.on('data',function(data){
                    buffer.push(data);
                });
                 //用于监听 end 事件 完成数据的接收
                res.on('end',function(){
                    result = Buffer.concat(buffer).toString('utf-8');
                    resolve(result);
                })
            })
            //监听错误事件
            .on('error',function(err){
                reject(err);
            });
            //传入数据
            req.write(data);
            req.end();
        });
    }
}

/**
 * 获取微信 access_token
 */
WeChat.prototype.getAccessToken = function(){
    var that = this;
    return new Promise(function(resolve,reject){
        //获取当前时间 
        var currentTime = new Date().getTime();
        //格式化请求地址
        var url = util.format(that.apiURL.accessTokenApi,that.apiDomain,that.appID,that.appScrect);
        //判断 本地存储的 access_token 是否有效
        if(accessTokenJson.access_token === "" || accessTokenJson.expires_time < currentTime){
            that.requestGet(url).then(function(data){
                var result = JSON.parse(data); 
                if(data.indexOf("errcode") < 0){
                    accessTokenJson.access_token = result.access_token;
                    accessTokenJson.expires_time = new Date().getTime() + (parseInt(result.expires_in) - 200) * 1000;
                    //更新本地存储的
                    fs.writeFile('./wechat/access_token.json',JSON.stringify(accessTokenJson));
                    //将获取后的 access_token 返回
                    resolve(accessTokenJson.access_token);
                }else{
                    //将错误返回
                    resolve(result);
                } 
            });
        }else{
            //将本地存储的 access_token 返回
            resolve(accessTokenJson.access_token);  
        }
    });
}
/**
 * 获取微信 jsapi_ticket
 */
WeChat.prototype.getJsapiTicket =function(access_token){
    var that = this;
    return new Promise(function(resolve,reject){
        //获取当前时间 
        var currentTime = new Date().getTime();
        //格式化请求地址
        var url = util.format(that.apiURL.jsapiTicketApi,that.apiDomain,access_token);
        //判断 本地存储的 jsapi_ticket 是否有效
        if(jsapiTicketJson.jsapi_ticket === "" || jsapiTicketJson.expires_time < currentTime){
            that.requestGet(url).then(function(data){
                var result = JSON.parse(data); 
                if(result.errcode == 0 && result.errmsg=='ok'){
                    jsapiTicketJson.jsapi_ticket = result.ticket;
                    jsapiTicketJson.expires_time = new Date().getTime() + (parseInt(result.expires_in) - 200) * 1000;
                    //更新本地存储的
                    fs.writeFile('./wechat/jsapi_ticket.json',JSON.stringify(jsapiTicketJson));
                    fs.writeFile('./wechat/access_token.json',JSON.stringify(accessTokenJson));
                    //将获取后的 jsapi_ticket 返回
                    console.log(jsapiTicketJson.jsapi_ticket)
                    resolve(jsapiTicketJson.jsapi_ticket);
                }else{
                    //将错误返回
                    resolve(result);
                } 
            });
        }else{
            //将本地存储的 jsapi_ticket 返回
            resolve(jsapiTicketJson.jsapi_ticket);  
        }
    });
}
// 创建随机字符串
WeChat.prototype.createNonceStr = function(){
  return Math.random().toString(36).substr(2, 15);
}
WeChat.prototype.createTimestamp = function () {
  return parseInt(new Date().getTime() / 1000) + '';
};
// 格式化字符串
WeChat.prototype.raw=function (args) {
  var keys = Object.keys(args);
  keys = keys.sort()
  var newArgs = {};
  keys.forEach(function (key) {
    newArgs[key.toLowerCase()] = args[key];
  });

  var string = '';
  for (var k in newArgs) {
    string += '&' + k + '=' + newArgs[k];
  }
  string = string.substr(1);
  return string;
};
WeChat.prototype.sign = function (jsapi_ticket, url) {
  var ret = {
    jsapi_ticket: jsapi_ticket,
    nonceStr: this.createNonceStr(),
    timestamp: this.createTimestamp(),
    url: url
  };
  var str = this.raw(ret);
  var hash =crypto.createHash('RSA-SHA1');
      hash.update(str);
      ret.signature=hash.digest('HEX');
    ret.appId=this.appID;

  return ret
}
//暴露可供外部访问的接口
module.exports = WeChat;
```
