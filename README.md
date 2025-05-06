# wework-message-async
(本项目改写自：Garfield-yin/wework-chat-node 感谢原作者)
由于原项目包长时间未维护，故重新修改立项，支持了 node.js 18+ 以上 目前测试至 node v22 可以正常使用；
同时修复了部分bug,更新了最新的企业微信 C语言 SDK v3.0 （2025-2-13）。
后续如果需要升级企业微信 SDK（当企微SDK有更新时：https://developer.work.weixin.qq.com/document/path/91774）,请更新 lib/libWeWorkFinanceSdk_C.so 以及 include/wework/WeWorkFinanceSdk_C.h （wget https://sdk文件下载路径   解压缩后手动替换相应文件），文件替换后再 build （npm rebuild）。
本模块也会持续更新优化。
使用中遇到问题，欢迎提交 issue。
也可微信联系我：realyanglin
##### Compiling

由于企业微信提供的 sdk 仅支持 linux 与 windows,在 OS X 下可编译成功，但无法正常使用。

[企业微信获取会话内容文档链接]https://work.weixin.qq.com/api/doc/90000/90135/91774

### Installation
1.请先安装 node-gyp
```
npm install node-gyp -g
```
2.然后安装本模块
```
npm install wework-message-async
```

3.使用示例
注意私钥格式，经常因为格式问题出错，私钥中间部分中间不要有换行，否则会解密失败

4.常见问题
关于seq，因为企微只能获取最近5天的消息，比如今天是10月5日，传0时即代表获取的是倒推5天的第一条消息（相对的），后续再获取的话就可以根据返回的最大seq来作为起始位置（绝对的），所以首次获取都会先用0作为起始位置，后续再获取时传入上次返回的last_seq即可。

关于私钥private_key，如果你设置的公钥版本只有1个，只要配置好对应的私钥用来解密即可，经常出问题的是存在多个版本的密钥时，比如设置过3个密钥，企微后台最新的公钥版本是3，你可能以为私钥也设置最新的就可以了，但获取消息记录的时候，加密的消息有个字段是公钥版本号publickey_ver ，你必须用对应的私钥才能正确解密 比如是3就用3对应的私钥 是1就要用1对应的私钥，除非你不考虑获取以前的消息了 只获取从设置最新密钥后的消息的话可以不用管它，但你需要先知道起始seq,否则就要等5天后再从seq:0 获取起始位置。
当然本模块暂时还未支持多个版本私钥的配置，后续会支持。
### Example

```javascript
import fs from "fs";
import {
	GetMediaDataParams,
	GetDataParams,
	WeWorkChat,
	ChatDataItem,
} from "wework-message-async";

const privateKey =
	"-----BEGIN RSA PRIVATE KEY-----\n" +
	"xxxxxxxxxxxxxxxxxxxxxxxxxxxx\n" +
	"-----END RSA PRIVATE KEY-----\n";

const wework = new WeWorkChat({
	/** 企业ID */
	corpid: "corpid",
	/** Secret */
	secret: "secret",
	/**私钥，用于消息解密 */
	private_key: privateKey,
	/** 数据拉取index */
	seq: 0,
});

// 获取媒体文件
const getMediaData = (
	fileName: string,
	params: GetMediaDataParams,
	bufs: Buffer[] = []
) => {
	const resp = wework.getMediaData(params);
	const bufVal = Buffer.from(resp.data);
	bufs.push(bufVal);
	if (!resp.is_finished) {
		// 分片读写,为了防止大文件 buffer 撑爆，建议使用 stream append 方式写文件
		params.index_buf = resp.buf_index;
		getMediaData(fileName, params, bufs);
	} else {
		const bufVal = Buffer.concat(bufs);

		fs.createWriteStream(fileName).write(bufVal);
	}
};

// 获取会话数据
const test = () => {
	const params: GetDataParams = {
		max_results: 10,
		timeout: 30,
		seq: 0,
	};
	const ret = wework.getChatData(params);
	console.log(ret.last_seq); //获取最后一条消息的seq 应保存起来下次可以传入从此开始获取 入库时建议根据msgid作唯一性检查
	for (const msg of ret.data) {
		if (!msg) continue;
		const msgData: ChatDataItem = JSON.parse(msg); //解析消息JSON
		if (msgData.msgtype != "file") continue; //根据消息类型判断
		if (msgData.file && msgData.file.fileext != "pptx") continue;
		const fileInfo = msgData.file;
		if (!fileInfo) continue;
		getMediaData(fileInfo.filename, {
			sdk_fileid: fileInfo.sdkfileid,
			index_buf: "",
		});
	}
};

test(); //执行
```

### TO DO
