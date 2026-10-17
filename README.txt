1.说明

这是一个把用户的一句话抽取成结构化事实的中文小模型；输出严格 JSON 行，只做"抽事实"这一件事，不做对话
一般用于辅助Character.AI 类、AI 伴侣、视觉小说 NPC、桌宠、语音助手的人格记忆
本模型完全在本地运行，不联网。您的私密对话、信息不会上传到云服务器

基座：Qwen2.5-1.5B-Instruct    量化：Q8_0

文件：fact-r8-q8_0.gguf（模型本体）
	 system_prompt.txt（系统提示词，不可更换，否则指标会崩）
	 SHA256SUMS.txt（校验码）
	 README.txt（本文件）

2.用法

推荐使用llama.cpp（也支持Ollama，可自行选择）

示例：
先确认llama-server可用
起服务：
llama-server -m fact-r8-q8_0.gguf --host 127.0.0.1 --port 8091 -c 2048 -np 1 -ngl 99 --alias r8

调用（OpenAI 兼容）：
注意：一次请求发一句话，不要成批输入
curl http://127.0.0.1:8091/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "r8", "temperature": 0, "max_tokens": 256,"repeat_penalty": 1.0,
  "messages": [
    {"role": "system", "content": "<system_prompt.txt 全文>"},
    {"role": "user",   "content": "<你要输入的原文>"}
  ]
}'

注意：调用时temperature一律建议手动设为0，否则模型会明显退化
	 max_tokens设256够用，经测试99%的输出在60token内
	 repeat_penalty建议设为1.0，调高可能会出现坏json（若你的客户端不认这个字段，可在启动参数里加 --repeat-penalty 1.0，两种写法等效。）
	 repeat_penalty 为 llama.cpp 扩展参数，换到其它后端时删除该项即可

3.输出格式

一行一个 JSON 对象。共三种形态：

// 形态 1：不自足（指代 / 省略 / 确认词 / 废话）
{"has_info": false, "self_contained": false}

// 形态 2/3：自足但没有关于"我"的可记信息（第三人称、反问、评价、玩笑…）
{"has_info": false, "self_contained": true, "certainty": "explicit"}

// 正例：全部字段
{"has_info": true, "self_contained": true, "subject_raw": "我",
 "predicate": "对", "object_raw": "花生过敏", "polarity": 1,
 "kind": "attribute", "certainty": "explicit", "time_expression": null}


字段：has_info, self_contained, subject_raw, predicate, object_raw, polarity, kind, certainty, time_expression

4.评测

测试时采用真实聊天记录289条（大量指代不明，省略）+手写句子100条（大多是完整句子）
r8：
真实记录289，含27条正事实：json合法率100%，负事实判正率0.8%（2条），正事实丢失率25.9%（7条），字段准确率95.7%
手写句子100，含56条正事实：json合法率100%，负事实判正率0%（0条），正事实丢失率5.4%（3条），字段准确率98.9%
本地Vulkan，单句延迟≈0.65s，单句成本0

对比DeepSeekv4.1flash（相同提示词，额外提供1419字符规则文本）：
真实记录289，含27条正事实：json合法率100%，负事实判正率14.1%（37条），正事实丢失率7.4%（2条），字段准确率86.9%
手写句子100，含56条正事实：json合法率100%，负事实判正率31.8%（14条），正事实丢失率1.8%（1条），字段准确率85.5%
单句延迟≈1.0s（含网络），单句成本约 800 tokens

对比Qwen3.8Max（相同提示词，额外提供1419字符规则文本）：
真实记录289，含27条正事实：json合法率100%，负事实判正率7.3%（19条），正事实丢失率22.2%（6条），字段准确率88.4%
手写句子100，含56条正事实：我没跑，太贵了（目移）跑完289条就花了十块多
单句延迟≈10.8s（含网络），单句成本约 1400 tokens


5.已知限制

中文为主，支持简单英文（但未系统实测）
设计为“只记录与说话人直接相关的、有记录价值的事实，拿不准就判负”，尽量保持数据入库时“宁可漏不可错”
为了方便抽取，设计为只抽取句子中出现的原话（除部分句式可以补“是”“有”来保证谓语行非空缺），但经过实测，模型仍有极小概率自己造字，不可完全信赖。使用时需要有后端逻辑兜底
本模型每次只抽取第一个关于说话人本身的事实分句，多事实句会舍弃超出第一个的部分，如有需要，请将句子切分后再喂给模型
在第一人称自述上训练：第三人称、反问、元话语、纯情绪会被判负，这是设计而非缺陷。

6.许可

权重与基座同为 Apache-2.0（Qwen2.5-1.5B-Instruct LICENSE：https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct/blob/main/LICENSE）
训练数据为自建中文语料，不含基座模型输出以外的第三方数据。

7.字段含义

"has_info": {"type": "boolean", "desc": "这句话里有没有关于“我”的可记事实"},
"self_contained": {"type": "boolean", "desc": "单拿出来看不看得懂（看不懂=不自足）"},
"subject_raw": {"type": "string", "desc": "只写第一人称（我/我们/咱们…），不还原、不补全"},
"predicate": {"type": "string", "desc": "从原句取；必要时补“是”“有”；体标记（了/过/着）紧跟谓语就归谓语"},
"object_raw": {"type": "string|null", "desc": "谓语后到第一个分句标点为止；没有就 null"},
"polarity": {"type": "integer", "enum": [1, -1],"desc": "肯定为1，否定为-1"},
"kind": {"type": "string", "enum": ["preference", "attribute", "event", "commitment", "relationship"]},
"certainty": {"type": "string", "enum": ["explicit", "speculation", "joke"]},
"time_expression": {"type": "string|null", "desc": "时间/频率的原话，不归一化"}

kind字段解释：
preference：偏好，喜好
attribute：状态，习惯，属性，特质
event：做过的事情
commitment：将要做的事情
relationship：拥有，持有关系
注意：很多句子的kind比较特殊，无法准确归类，因此该字段只能用作参考

8.简单单句示例

in : 我对花生过敏
out: {"has_info": true, "self_contained": true, "subject_raw": "我", "predicate": "对","object_raw": "花生过敏", "polarity": 1, "kind": "attribute","certainty": "explicit", "time_expression": null}

in : 我有个弟弟
out: {"has_info": true, "self_contained": true, "subject_raw": "我", "predicate": "有","object_raw": "个弟弟", "polarity": 1, "kind": "relationship","certainty": "explicit", "time_expression": null}

in：我不太喜欢公司的氛围
out：{"has_info": true, "self_contained": true, "subject_raw": "我", "predicate": "喜欢", "object_raw": "公司的氛围", "polarity": -1, "kind": "preference", "certainty": "explicit", "time_expression": null}

in : 你连这都知道，真牛逼啊
out: {"has_info": false, "self_contained": false}

in : 不，我的意思是，你知道这意味着什么吗
out: {"has_info": false, "self_contained": false}

in : 我昨天还想今天去医院来着，不过今天突然说是有课
out: {"has_info": false, "self_contained": true, "certainty": "explicit"}