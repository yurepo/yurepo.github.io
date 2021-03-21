---
title: "如何將Hugo部落格部署到Github上?"
date: 2021-03-21T20:47:53+08:00
draft: false
---

# 手把手教學: 將Hugo部落格佈署到Github上

## 前言

最近將以前的部落格從Hexo遷移到Hugo上了，不得不說Hugo在產生靜態頁面的速度比起Hexo來說快了很多，得益於Hexo跟Hugo都是使用Markdown文件的原因，在兩者之間進行遷移是非常容易的，今天就來為自己做個筆記，希望大家看了這篇文章後，沒有部落格的都可以嘗試建立一下自己的部落格。

## 教學開始

這邊安裝的Hugo版本為`hugo v0.81.0`，環境為Win10 20H2 x64，使用的工具為[git](https://git-scm.com/downloads)與[chocolatey](https://chocolatey.org/install)，若是macOS，則可以使用[Homebrew](https://brew.sh/index_zh-tw)，Linux的部分則在這邊[下載](https://docs.brew.sh/Homebrew-on-Linux)

### 第一步，安裝Hugo

這邊主要就照著[官方的快速開始](https://gohugo.io/getting-started/quick-start/)建立部落格，因為官方已經寫的足夠清楚了，所以在此處僅寫Windows版本的安裝過程

如果要安裝普通版本的Hugo，請使用以下指令

```powershell
choco install hugo -confirm
```

如果要安裝Sass/SCSS版本的，請使用以下指令(有些主題會要求需要hugo-extended版本)

```powershell
choco install hugo-extended -confirm
```

### 第二步，建立部落格

安裝完Hugo後，可以使用`hugo version`確認是否安裝成功。

安裝成功後，在你想要建立部落格的資料夾內打開powershell，並打以下指令即可建立。

```powershell
hugo new site <資料夾名稱>
```

出現以下畫面就說明安裝成功了!

![圖像](/img/hugo-create-new-blog.gif)

### 第三步，添加主題

接下來到[官方的主題網站](https://themes.gohugo.io/)挑選您喜歡的主題，此範例使用[ananke](https://github.com/budparr/gohugo-theme-ananke.git)主題。

只需要打以下指令就可以新增主題了

```powershell
cd <資料夾名稱>
git init
git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
```

接著修改`config.toml`文件，詳細的設定方式可以參照[官網](https://gohugo.io/getting-started/configuration/)

```toml

# 基本設置
baseURL = "<網址>"
title = "<標題>"
languageCode = "en-us"

# 主題設置
theme="ananke"

# 連結設置
[permalinks]
  posts = "/:year/:month/:title/"

```

### 第四步，建立第一篇貼文

接著輸入以下指令建立第一篇貼文

```powershell
hugo new posts/hello-world.md
```

接著打開Markdown編輯工具(e.g. Visual Studio Code)，寫點簡單的文章並存檔。

```markdown
---
title: "Hello World"
date: 2021-03-21T21:46:28+08:00
draft: false
---
# Hello world

我在`2021-03-21`建立了第一篇文章。

```

### 第五步，預覽網站

就跟Hexo一樣，Hugo也提供了本地伺服器的功能，僅需在部落格資料夾下使用powershell或cmd打`hugo server -D`就可以在本地端預覽網站，預設網址為: `http://localhost:1313/`

### 第六步，將Hugo部落格放到Github上

接下來，在github建立一個存放網站用的Repository，並將其命名為

`<username>.github.io`

註: username必須是您在Github上的的使用者名稱

![圖像](/img/github-create-hugo-repository.PNG)

接下來，在部落格資料夾下建立一條`gh-pages`分支(branch)，這個分支是用來展示靜態頁面的，我們稍後會使用Github Action將主分支的內容透過自動化部署的方式，自動產生靜態文件到`gh-pages`分支上。

```powershell

# 加入所有檔案
git add .
# 新增commit內容
git commit -m "init blog"
# 新增main分支
git branch -M main
# 新增遠端版本庫
git remote add origin https://github.com/<使用者名稱>/<使用者名稱>.github.io.git
# 將部落格內容上傳到remote
git push -u origin main

# 新增gh-pages孤兒分支
git checkout --orphan gh-pages
# 砍掉gh-pages分支的所有檔案
git rm -rf .
# 新增一個README.md檔
echo "gh-pages" > "README.md"
# 加入所有檔案
git add .
# 新增commit內容
git commit -m "init gh-pages branch"
# 將分支內容上傳到remote
git push -u origin gh-pages
# 切換到main分支
git checkout main

```

將上面的指令打完後，照理來講會出現兩個分支，一個叫做`main`，另一個叫做`gh-pages`

### 第七步，設定Github Action進行自動化部署

要怎麼將main分支上的hugo檔案，自動化部署到gh-pages分支上呢?

#### 前置作業

首先我們先到[這個頁面](https://github.com/settings/tokens/new)去取得我們的Personal Access Token (等等會用到)。

將`workflow`那一項打勾之後到頁面最下方按下`Generate Token`

![圖像](/img/github-hugo-personal-key.PNG)

將產生出來的令牌先複製起來

![圖像](/img/github-hugo-get-personal-key.PNG)

接下來到你存放Hugo部落格的Repository > Settings > Secret > New repository secret新增你剛取得的令牌

![圖像](/img/github-hugo-repository-secret.PNG)

將Name取為`HUGO_DEPLOY_TOKEN`，Value設定為剛取得的令牌，按下`Add Secret`

![圖像](/img/github-hugo-create-secret-token.PNG)

至此前置動作完成，接下來開始設定workflow

#### 設定workflow

我們將會參考[此文](https://github.com/peaceiris/actions-hugo)的workflow文件去設定Github Action。

首先，進到Github Action頁面，並點選`set up a workflow your self`

![圖像](/img/github-actions-hugo.PNG)

將下面的文件貼上並修改一些設定(e.g. name)

```yaml
name: <workflow名稱>

on:
  push:
    branches:
      - main  # 當main分支有push操作時

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # 找尋Hugo主題(true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.81.0' # hugo 版本
          # extended: true  # 如果是使用extended版本的務必取消註解。

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.HUGO_DEPLOY_TOKEN }}
          PUBLISH_BRANCH: gh-pages  # 推送到 gh-pages 分支
          PUBLISH_DIR: ./public     # hugo 生成的目錄
          commit_message: ${{ github.event.head_commit.message }}
```

打好後按下右上角的`Start commit`儲存完workflow的設定文件後就可以在Repository的Action頁面看到workflow的執行狀態

若執行狀態為綠色打勾即為部署成功!

![圖像](/img/github-hugo-action-deploy.PNG)

### 第八步，部落格建置

最後一步，到你存放Hugo部落格的Repository > Settings內，將GitHub Pages的分支改為`gh-pages`就大功告成了!

之後新增文章只需要三個步驟

1. 建立新的貼文
2. 撰寫貼文
3. 上傳整個hugo文件夾到github

剩下的靜態文件產生會由Github Action幫你處理並自動幫你部署到`gh-pages`分支上。

![圖像](/img/github-hugo-deploy-complete.PNG)

## 結語

至此只需要簡單幾個步驟就可以建立Hugo網站並將網站Host到Github Pages上了，希望可以幫助到想要建立自己部落格的人！
