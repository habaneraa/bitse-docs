# BITSE 课题组文档站

使用 Docker 预览站点

```bash
docker run --rm -it -p 3000:80 --name bitse-docs -v ${PWD}:/docs squidfunk/mkdocs-material
```

使用 Docker 构建

```bash
docker run --rm -it -v ${PWD}:/docs squidfunk/mkdocs-material build
```
