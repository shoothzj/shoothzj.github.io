---
title: 错误码国际化总结
date: 2024-03-21 16:50:29
tags:
  - Code
---

# 错误信息无模板变量

假设我们的错误信息返回如下

```text
HTTP/1.1 200 OK
{"error_code": "IEEE.754", "error_msg": "IEE 754 error"}
```

无模板变量的错误信息国际化，可以直接在前端对整体字符串根据错误码进行静态国际化。

```typescript
// catch the error code first
const error_code = body.error_code

const error_msg_map = {
    "IEEE.754": {
        "en": "IEE 754 error",
        "zh": "IEE 754 错误"
    }
}

const error_msg = error_msg_map[error_code][lang]
```

# 错误信息包含模板变量

假设我们的错误信息返回如下

```text
HTTP/1.1 200 OK
{"error_code": "IEEE.754", "error_msg": "IEE 754 NbN error, do you mean Nan?"}
```

包含模板变量的错误信息国际化，可以在前端通过正则表达式提取，并代入到中文字符串模板中实现。如示例代码

```typescript
// catch the error code first
const error_code = body.code

const error_msg_capture_map = {
    "IEEE.754": "/IEE 754 (\w+) error, do you mean (\w+)?/"
};

const error_msg_template_map = {
    "IEEE.754": {
        "en": "IEE 754 {{var1}} error, do you mean {{var2}}?",
        "zh": "IEE 754 {{var1}} 错误，你是指 {{var2}} 吗？"
    }
};

const matches = error_msg_capture_map[error_code].exec(body.error_msg);
const variables = matches.slice(1);

let error_msg = error_msg_template_map[error_code][lang];
variables.forEach((value, index) => {
    error_msg = error_msg.replace(`{{var${index + 1}}}`, value);
});
```
