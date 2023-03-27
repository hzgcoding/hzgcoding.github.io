## 项目目录
- scaffolds 
脚手架， 相当于模板

- source
文章源码，这里主要是markdown文件

- themes 
主题文件， 通过_config.yml 文件指定具体使用哪个主题

- _config.yml
配置文件

## Theme主题

```
git clone https://github.com/renbaoshuo/hexo-theme-pure.git themes/pure

```

## 安装

```
npm install -g hexo-cli
npm install hexo-deployer-git
npm install
```

## Hexo 
Usage: hexo <command>
```
Commands:
  clean     Remove generated files and cache.
  config    Get or set configurations.
  deploy    Deploy your website.
  generate  Generate static files.
  help      Get help on a command.
  init      Create a new Hexo folder.
  list      List the information of the site
  migrate   Migrate your site from other system to Hexo.
  new       Create a new post.
  publish   Moves a draft post from _drafts to _posts folder.
  render    Render files with renderer plugins.
  server    Start the server.
  version   Display version information.

Global Options:
  --config  Specify config file instead of using _config.yml
  --cwd     Specify the CWD
  --debug   Display all verbose messages in the terminal
  --draft   Display draft posts
  --safe    Disable all plugins and scripts
  --silent  Hide output on console
```