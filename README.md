### wxrecording(微信录音识别成文字 demo)

> 之前接到过这样一个需求（好气啊，需求在运营那里最后夭折了），立刻去查了微信的api，开始做了测试demo，遇到了坑，但也找到了优化方案，有问题欢迎拍砖~~~

#### 首要的重中之重：
1.使用过微信开发的伙伴们应该都知道，在使用微信的JS-SDK对应的JS接口前，需确保已获得使用对应JS接口的权限，不知道的也没有关系，[具体可以点击了解](http://qydev.weixin.qq.com/wiki/index.php?title=%E5%BE%AE%E4%BF%A1JS%E6%8E%A5%E5%8F%A3#.E8.AF.86.E5.88.AB.E9.9F.B3.E9.A2.91.E5.B9.B6.E8.BF.94.E5.9B.9E.E8.AF.86.E5.88.AB.E7.BB.93.E6.9E.9C.E6.8E.A5.E5.8F.A3)

2.其二还是权限问题，所有需要使用JS-SDK的页面必须先注入配置信息，而这些配置信息也是需要主动向微信申请的。有了这些信息我们就可以开始放纵了~~~

3.调用的JS-SDK几个接口，大概思路很简单：录音--->停止录音时拿到本地录音id--->通过本地录音id识别录音

#### 开始coding
##### 1.获取到配置信息，进行注入：

    wx.config({
	    debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
	    appId: '', // 必填，企业号的唯一标识，此处填写企业号corpid
	    timestamp: , // 必填，生成签名的时间戳
	    nonceStr: '', // 必填，生成签名的随机串
	    signature: '',// 必填，签名，见附录1
	    jsApiList: [] // 必填，需要使用的JS接口列表
    });
##### 2.给你的录音按钮绑定相关事件

    frecording: function(){
      var self = oPage;
      self.startTime = new Date().getTime();
      $('.J_doc').on('touchend', '.J-recordingBtn', function(){
             self.fstopRecording();
       });
       wx.ready(function(){
          wx.startRecord({
               success: function (res) {
                   localStorage.userAllowRecord = 'true';//记录用户是否允许录音
                   self.countTime();//疑点1  为什么调用它？后面讲述
                   self.fautoStopvoice();//疑点2  为什么调用它？后面讲述
               },
               cancel: function () {
                   self.toast('你已拒绝授权录音~');
               }
           });
        });
    }
    

>在这里有点说明：
>`1.`调用微信的语音，参与者需要手动允许，即要同意允许使用微信录音功能，要是不允许，录音就废掉了。
>
>但是为了流失用户，我采取了这样的**优化方案**：用户未允许，记录该状态，后面每次刚进入页面，自动录音，这时会唤起允许弹窗，引导用户允许

#### 3.停止录音

    fstopRecording: function(){
      var self = oPage,
          _timediff = new Date().getTime() - self.startTime;   
      if ($recordingBtn.hasClass('noNeedStop')) {
          return;
      }              
      wx.ready(function(){
          //说话时间太短
          if (_timediff <= 800){
              wx.stopRecord({
                  success: function (res) {
                      self.voiceId = res.localId;
                  }
              });
              self.toast('说话时间太短~');
          } else {
              wx.stopRecord({
                  success: function (res) {
                      self.voiceId = res.localId;
                      self.fDistinguishVoice(self.voiceId);//识别录音
                  },
                  fail: function (res) {
                      alert('fail');
                  },
                  complete: function () {
                      alert('complete');
                  }
              });
          }                
      });
	}

> 这里就涉及到一个问题：
> `1.`一次语音输入时间不可以超过1分钟,超过一分钟会自动停止！！！
> 于是就有了上面疑点1，疑点2处的函数调用~~~
>
> `2.self.countTime()`我用来记录时间，如果到了指定时间，我会给按钮加一个标记，防止录音超过60s已经自动停止，再次点击停止录音，又开始新的录音，当然如果开始录音与停止录音是两个不同的按钮，那么这步可以省略
>
> `3.self.fautoStopvoice()`因为录音超过60s会自动停止，而我需要识别录音，那么必须去监听这个事件以记录录音id

    fautoStopvoice: function(){
      var self = oPage;
      wx.ready(function(){
          wx.onVoiceRecordEnd({
              // 录音时间超过一分钟没有停止的时候会执行 complete 回调
              complete: function (res) {
                  self.voiceId = res.localId;
                  clearTimeout(self.timer);
                  self.fDistinguishVoice(self.voiceId);
              }
          });
      });
    }

#### 4.识别录音

    fDistinguishVoice: function(voiceId){
      var self = oPage;
      wx.translateVoice({
         localId: voiceId, // 需要识别的音频的本地Id，由录音相关接口获得
         isShowProgressTips: 1, // 默认为1，显示进度提示
          success: function (res) {
              $input.val().length == 0 ? $input.val(res.translateResult) : $input.val($input.val() + res.translateResult);
              $recordingBtn.removeClass('noNeedStop');
          },
          fail: function (res) {
              self.toast('翻译失败~');
          },
          complete: function () {
              self.toast('翻译完成~');
          }
      });
    }

> `这里也有尴尬的一点需要注意`：
>一次语音说完，才可以识别成文字，会有一个是识别中的toast（识别时间与网络以及语音长短有关），无法边说边识别

#### ps：有几点需要注意下
- 1.只能识别中文
- 2.第一次尝试，有问题欢迎交流~~~~
