# 暂存内容检查标准

本文件是项目级 Git 暂存内容检查标准。调用方启用预检时必须完整读取本文件，并仅检查 Git index 中的暂存版本，不得以工作区未暂存版本代替。

## 文件类型和读取规则

1. 识别新增、修改、删除、重命名、复制和二进制文件。
2. 删除文件不扫描删除内容；重命名同时检查旧路径和新路径的敏感文件名。
3. 二进制文件跳过文本内容扫描，但继续执行文件名和大小检查，并显示文件名、状态和大小。
4. 无法读取暂存 blob、无法识别编码或扫描器执行失败时停止当前操作，不把失败视为未发现问题。

## 大文件检测

large_file_size_limit 必须是正数，格式为数字加 B、KB、MB 或 GB，单位不区分大小写，按 1024 进位计算。例如 1MB 等于 1048576 字节。

- 暂存 blob 大小超过阈值：警告并询问是否继续；
- 所有非删除暂存 blob 总大小超过阈值的 10 倍：警告并询问是否继续；
- 使用 git check-attr filter -- PATH 检查实际路径是否配置 Git LFS；
- 如果匹配 LFS，提示应由 LFS 管理；不自动转换文件或修改 .gitattributes；
- 用户拒绝或取消时终止，保留当前暂存状态。

## 敏感文件名检查

按不区分大小写的完整路径或文件名匹配：

- 直接阻止：私钥文件（*.key、*.pem、*.p12、*.pfx、*.jks）、凭据文件（credentials*）、环境文件（.env 及其变体）、id_rsa、id_ed25519；
- 警告并确认：文件名包含 secret 或 password 的普通文件、*.pub 公钥文件；
- 删除敏感文件不按“新增敏感文件”阻止，但仍在报告中说明删除动作。

直接阻止时不提供“确认后继续”选项；警告级别必须由用户明确选择继续或终止。

## 敏感信息内容扫描

内容扫描必须针对 git show :PATH 输出的暂存版本，使用 rg --pcre2 -n -I 或等价的 PCRE2 扫描器。正则不得再经过 Markdown 表格转义。rg 返回 0 表示匹配，1 表示无匹配，2 或更高值表示扫描失败。执行器必须丢弃原始匹配行，只保留文件路径、行号和规则名称。

阻止级别模式：

    AWS Access Key: \b(?:AKIA|ASIA)[0-9A-Z]{16}\b
    AWS Secret Key: (?i)\b(?:aws_secret_access_key|aws_secret)\s*[:=]\s*['"]?[A-Za-z0-9/+=]{40}['"]?
    GitHub Token: \b(?:ghp|gho|ghu|ghs|ghr)_[A-Za-z0-9_]{36,}\b
    Private Key: -----BEGIN (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----
    Stripe Secret Key: \b(?:sk|rk)_(?:test|live)_[0-9A-Za-z]{10,}\b
    Slack Token: \bxox[bapors]-[0-9A-Za-z-]{10,}\b

警告级别模式：

    Generic API Key: (?i)\b(?:api[_-]?key|apikey)\s*[:=]\s*['"][0-9A-Za-z]{32,}['"]
    Password in Code: (?i)\b(?:password|passwd|pwd)\s*[:=]\s*['"][^'"]{8,}['"]
    Connection String: (?i)\b(?:mysql|postgres(?:ql)?|mongodb(?:\+srv)?)://\S{20,}
    JWT Token: \beyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\b
    Generic Secret: (?i)\b(?:secret|token)\s*[:=]\s*['"][0-9A-Za-z_-]{16,}['"]
    Stripe Publishable Key: \bpk_(?:test|live)_[0-9A-Za-z]{10,}\b

扫描结果只显示文件路径、行号和类型，不显示完整匹配值；不得直接向用户展示扫描器原始输出。

- 阻止级别：立即终止，不允许用户绕过；
- 警告级别：显示类型和位置，询问是否继续；
- 用户拒绝或取消：终止并保留暂存状态。
