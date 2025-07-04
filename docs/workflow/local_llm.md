# 本地大模型部署

为了方便课题组使用开源 LLM，充分利用服务器资源，我们在服务器上长期部署了 [Ollama](https://ollama.com/) + [Open WebUI](https://github.com/open-webui/open-webui) 实例。

- Ollama 实例位于 11434 端口，可使用 CLI `ollama` 直接使用，或通过 HTTP 访问裸 API
- OpenWebUI 实例位于 8001 端口，可使用浏览器直接访问 (注意使用 http 而非 https)

经同学反馈，模型部署有时会出现永久卡死的情况，在此提供一些解决方案：

1. 请检查是否限制了 LLM 生成长度 (最大输出 token 数) 如果留空则可能导致弱模型的无限生成；
2. 必要时请重启 Ollama 服务 `systemctl restart ollama`
3. 仍然不能解决请通过 `journalctl -u ollama --no-pager --follow --pager-end` 查看后台日志

## Open-WebUI 部署方法

直接使用 Docker 即可，镜像名称那里可以换成最新版本。该容器会默认访问 http://localhost:11434 也就是 Ollama 实例。

```bash
docker run -d \
  -p 8001:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui-new:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:v0.6.15
```
