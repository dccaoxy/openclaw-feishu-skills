# feishu-doc-manager

📄 飞书文档管理器 | Feishu Doc Manager

通过 API 创建、写入和管理飞书云文档，无需用户 OAuth 授权。

---

## 核心能力

- ✅ 使用 **tenant_access_token**（应用身份）创建文档，**无需用户授权**
- ✅ 写入文本内容（逐个 block 写入）
- ✅ 添加文档协作者权限
- ⚠️ 批量写入有 bug，需要逐个写入 blocks

---

## 工作流程

### 1. 获取 tenant_access_token

```python
import urllib.request
import json

app_id = "你的app_id"
app_secret = "你的app_secret"

token_url = "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal"
token_data = {"app_id": app_id, "app_secret": app_secret}
req = urllib.request.Request(
    token_url,
    data=json.dumps(token_data).encode(),
    headers={"Content-Type": "application/json"},
    method="POST"
)
with urllib.request.urlopen(req, timeout=10) as resp:
    tenant_token = json.loads(resp.read()).get("tenant_access_token", "")
```

### 2. 创建文档

```python
def create_doc(tenant_token, title, folder_token=None):
    url = "https://open.feishu.cn/open-apis/docx/v1/documents"
    data = {"title": title}
    if folder_token:
        data["folder_token"] = folder_token
    
    req = urllib.request.Request(
        url,
        data=json.dumps(data).encode(),
        headers={
            "Authorization": f"Bearer {tenant_token}",
            "Content-Type": "application/json"
        },
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        result = json.loads(resp.read())
        doc_id = result["data"]["document"]["document_id"]
        return doc_id
```

### 3. 写入内容（逐个 block）

```python
def add_text_block(tenant_token, doc_id, content, block_type=2):
    """block_type: 2=text, 4=heading1, 12=bullet"""
    block_url = f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children"
    
    if block_type == 2:
        block_data = {
            "block_type": 2,
            "text": {
                "elements": [{"text_run": {"content": content, "text_element_style": {}}}],
                "style": {}
            }
        }
    elif block_type == 4:
        block_data = {
            "block_type": 4,
            "heading1": {
                "elements": [{"text_run": {"content": content, "text_element_style": {}}}],
                "style": {}
            }
        }
    elif block_type == 12:
        block_data = {
            "block_type": 12,
            "bullet": {
                "elements": [{"text_run": {"content": content, "text_element_style": {}}}],
                "style": {}
            }
        }
    
    req = urllib.request.Request(
        block_url,
        data=json.dumps({"children": [block_data]}).encode(),
        headers={
            "Authorization": f"Bearer {tenant_token}",
            "Content-Type": "application/json"
        },
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.loads(resp.read())
```

### 4. 添加协作者

```python
def add_collaborator(tenant_token, doc_token, user_open_id, perm="full_access"):
    """perm: full_access, edit, view"""
    url = f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx"
    data = {
        "member_type": "openid",
        "member_id": user_open_id,
        "perm": perm
    }
    req = urllib.request.Request(
        url,
        data=json.dumps(data).encode(),
        headers={
            "Authorization": f"Bearer {tenant_token}",
            "Content-Type": "application/json"
        },
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        return json.loads(resp.read())
```

---

## 完整示例

```python
import urllib.request
import json

def feishu_create_doc_with_content(app_id, app_secret, user_open_id, title, content_blocks):
    """
    创建飞书文档并写入内容
    
    content_blocks: list of tuples [(block_type, content), ...]
    block_type: 2=text, 4=heading1, 12=bullet, 13=ordered
    """
    
    # 1. 获取 token
    token_url = "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal"
    req = urllib.request.Request(
        token_url,
        data=json.dumps({"app_id": app_id, "app_secret": app_secret}).encode(),
        headers={"Content-Type": "application/json"},
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        tenant_token = json.loads(resp.read())["tenant_access_token"]
    
    # 2. 创建文档
    doc_url = "https://open.feishu.cn/open-apis/docx/v1/documents"
    req = urllib.request.Request(
        doc_url,
        data=json.dumps({"title": title}).encode(),
        headers={
            "Authorization": f"Bearer {tenant_token}",
            "Content-Type": "application/json"
        },
        method="POST"
    )
    with urllib.request.urlopen(req, timeout=10) as resp:
        doc_id = json.loads(resp.read())["data"]["document"]["document_id"]
    
    # 3. 写入内容
    for block_type, content in content_blocks:
        block_url = f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children"
        
        if block_type == 2:
            block_data = {"block_type": 2, "text": {"elements": [{"text_run": {"content": content, "text_element_style": {}}}], "style": {}}}
        elif block_type == 4:
            block_data = {"block_type": 4, "heading1": {"elements": [{"text_run": {"content": content, "text_element_style": {}}}], "style": {}}}
        elif block_type == 12:
            block_data = {"block_type": 12, "bullet": {"elements": [{"text_run": {"content": content, "text_element_style": {}}}], "style": {}}}
        elif block_type == 13:
            block_data = {"block_type": 13, "ordered": {"elements": [{"text_run": {"content": content, "text_element_style": {}}}], "style": {}}}
        else:
            continue
        
        req = urllib.request.Request(
            block_url,
            data=json.dumps({"children": [block_data]}).encode(),
            headers={
                "Authorization": f"Bearer {tenant_token}",
                "Content-Type": "application/json"
            },
            method="POST"
        )
        try:
            urllib.request.urlopen(req, timeout=10)
        except:
            pass
    
    # 4. 添加权限
    perm_url = f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_id}/members?type=docx"
    req = urllib.request.Request(
        perm_url,
        data=json.dumps({"member_type": "openid", "member_id": user_open_id, "perm": "full_access"}).encode(),
        headers={
            "Authorization": f"Bearer {tenant_token}",
            "Content-Type": "application/json"
        },
        method="POST"
    )
    try:
        urllib.request.urlopen(req, timeout=10)
    except:
        pass
    
    return doc_id


# 使用示例
app_id = "cli_xxx"
app_secret = "xxx"
user_open_id = "ou_xxx"  # 曹晓雨的 open_id
doc_id = feishu_create_doc_with_content(
    app_id, app_secret, user_open_id,
    title="测试文档",
    content_blocks=[
        (4, "标题"),
        (2, "正文内容"),
        (12, "列表项1"),
        (12, "列表项2"),
    ]
)
print(f"文档创建成功: https://sample.feishu.cn/docx/{doc_id}")
```

---

## 注意事项

1. **批量写入有 bug**：一次发送多个 blocks 会返回 400 错误，需要逐个写入
2. **tenant_access_token 有有效期**：2小时，过期需要重新获取
3. **文档 URL**：`https://sample.feishu.cn/docx/{doc_id}`，把 `sample` 换成实际域名
4. **权限类型**：`full_access`(完全管理), `edit`(编辑), `view`(查看)

---

## 相关资源

- [飞书创建文档 API](https://open.feishu.cn/document/server-docs/docs/docs/docx-v1/document/create)
- [飞书写入 blocks API](https://open.feishu.cn/document/server-docs/docs/docs/docx-v1/document-block/batch-create)
- [飞书权限管理 API](https://open.feishu.cn/document/server-docs/drive-v1/permission-member/create)
