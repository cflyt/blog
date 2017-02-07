---
title: Yeoman前端自动化构建
tags: 技术
categories: 未分类
date: 2017-02-07 10:40:38
type:
---

## Yeoman前端自动化构建
作为一名长期习惯在后端捣鼓的web开发人员，要兼顾前端的开发时，对于前端组织和优化往往不得要领，需要学习前端的知识来提高开发效率和优化web性能。目前我所接触到的web项目并不能完全前后端分离（后端只提供接口给前端），基本都是后端程序员将数据填进页面模板加以渲染，所以本文着重前端静态资源的自动化构建部分。

Yeoman是一套针对Web应用开发流程进行管理的工具集，它将前端无序、繁杂的操作组织起来，利用工具简化、规范前端流程，实现项目构建、开发、维护的一体化。用到的工具主要有以下3个：

* yo：脚手架工具；
* bower： 包管理工具；
* glup（grunt升级版）：构建工具；

这三个工具是分别独立开发的，但是需要配合使用，来实现我们高效的工作流模式。

Yeoman的项目中工作流程一般是：

1. yo创建项目骨架：运行yo工具，通过各种yeoman-generator（模板）创建项目骨架;
2. bower下载前端资源：运行bower install，下载项目中依赖的前端资源，比如jQuery、bootstrap、angularjs等；
3. gulp运行构建任务：运行grunt跑自动化构建任务等；


### nodejs安装
在使用yeoman工具之前需要先安装nodejs，nodejs目前没有apt方式安装，需要用源码编译安装。
从github上下载源码,根据tag选择需要的版本，我选择的是0.12.0:

    git clone https://github.com/joyent/node.git
    git checkout v0.12.0

编译安装：

    ./configure
    make
    sudo make install
安装好后，npm包管理工具也在其中了。

**ps**： npm默认的安装源是国外的，下载速度很慢，可以在用户目录下配置国内镜像源：

    registry = http://r.cnpmjs.org/

保存为.npmrc。

然后安装Yeoman的3个工具：

    sudo npm install -g yo bower gulp


### yo
yo是项目脚手架工具，在创建项目的时候使用，它帮你搭起项目的骨架，比如生成项目的目录结构，配置常用的插件，提供包括代码校验，测试，压缩 的项目流程。这好比IDE工具，eclipse，xcode等，创建一个项目时，帮你生成一些文件和目录，搭起项目雏形，只不过yo没有提供界面。

yo提供了非常多的模板，用来生成不同类型的 Web 应用。这些模板称为生成器（generator）。这些模板以一个工程化标准方式在项目初期阶段就建立好良好的代码结构和流程习惯，整合了业界的最佳实践。
generator-webapp是其中一个最典型的web项目模板，使用前需要npm先安装：

    npm install -g generator-webapp
然后用yo进行安装：

    mkdir mywebapp
    cd mywebapp
    yo webapp   #使用generator-webapp生成器模板

创建过程中会让你选择sass，bootstrap这些组件，根据项目需要选择，我选择了bootstrap，得到目录结构如下：

    .
    |-- app
    |   |-- robots.txt
    |   |-- index.html
    |   |-- styles
    |   |    |
    |   |    `-- main.css
    |   `-- scripts
    |        |
    |        `-- main.js
    |
    |-- bower.json
    |-- bower_components
    |   |-- bootstrap
    |   |-- jquery
    |   `-- ....
    |
    |-- gulpfile.babel.js
    |-- package.json
    |-- node_modules
    |   |-- gulp
    |   |-- gulpuglify
    |   `-- ....
    |
    `-- test
       |-- index.html
       `-- spec

下面对目录进行简单介绍：

* app：源文件的存放目录,未压缩的js,css,scss等;
* bower.json：bower的配置文件，根据配置安装第3方组件到bower_components文件夹下;
* bower_components：使用bower安装的第3方组件的源代码,jquery,bootstrap等;
* package.json: gulp的配置文件，根据配置安装gulp的依赖组件到node_modules文件夹下
* node_modules：gulp需要用到的工具模块安装目录;
* gulpfile.babel.js: gulp的任务配置文件，根据任务配置自动化处理css,js等；
* test 单元测试存放目录；

整个目录都很清晰，大致就是三部分：源文件，bower第三方库，gulp构建部分；

### bower
bower作为一个前端包依赖管理的工具，它可用于搜索、安装和卸载如JavaScript、HTML、CSS之类的网络资源。类似于java中maven，python中pip工具；
使用bower可以让你轻松的引入一个前端所需要的包，帮你解决包之间的依赖关系，比如你项目中会用到了bootstrap, jquery, jquery.ui. 这三个库都用到了jquery，所以你必须要去找到一个合适的jquery版本来让bootstrap和jquery.ui都可以正常工作，使用bower这些都不需要你来操心，一条命令即可。

#### bower使用
bower依赖node，npm，git工具，使用bower需要先安装好这些工具。

执行命令：

    bower install jquery

在当前目录下自动创建`bower_components`文件，里面存放了jquery部件。

##### bower.json
还可以通过配置 `bower.json` 文件，配置需要的安装的库以及版本等，该文件可以通过命令：

    bower init

创建符合bower规范的包配置文件，有了这个文件，直接执行`bower install`命令，bower自动在当前目录下搜索该文件，并自动安装已经配置的包文件。以后安装其他库加上`--save`参数，bower安装完该库后，把信息添加到bower.json文件中，然后把这个bower.json文件传给其他人，也是一条命令就搞定的事。

##### .bowerrc
bower 支持一个 `.bowerrc` 的配置，可以放置在当前目录下。就像yo生成的webapp项目下面就有这个文件。可以配置默认的安装目录directory,下载代理proxy,https-proxy等。
详见官方文档
https://github.com/bower/spec/blob/master/config.md 。

##### bower其他命令
* bower list 列出当前的安装包，以及包之间的依赖关系；

        bower list
        ├─┬ bootstrap#3.3.6 (latest is 4.0.0-alpha.2)
        │ └── jquery#2.2.1 (latest is 3.0.0-beta1)
        ├── chai#3.5.0
        └── mocha#2.4.5

*   bower uninstall 删除包
*   bower info 查看包的信息

yo脚手架已经将bower合进来了，只要在项目目录下进行上面介绍的操作就行了。

### gulp
最成熟和强大的要数这个gulp工具了，它不仅能对网站资源进行优化，而且能帮我们完成前端开发过程中的很多重复的任务用来运行各种任务，比如文件压缩、合并、打包等；项目生成、预览和自动化测试。

gulp本身不具备很多功能，主要依赖于一系列的功能插件，是一个基于流的构建工具；gulp构建的任务就是把合适的插件组装起来，像一条自动化的流水线一样，前一级的的输出变成后一级的输入，完成压缩，合并等复杂的功能。

gulp的全局安装前面介绍过了，需要通过npm包安装（前提是已经安装了nodejs环境 和 npm）。

####package.json
gulp的插件是由npm来管理的，由npm来安装。和bower的bower.json文件一样，npm使用`package.json`配置和管理所需的插件。实际上bower也是npm的一个模块。
npm安装的插件放置在当前目录的node-moduels目录中，package.json可以由命令：

    npm init
创建，可以在该文件中配置依赖的插件和版本，运行「npm install」命令，自动在当前目录下寻找package.json文件安装插件。
此外，安装其他插件加上`--save-dev`, npm安装完后将插件信息添加到package.json中。

#### gulpfile.js
gulp任务的构建需要依赖一个js文件`gulpfile.js`，而我们的项目中的却是gulpfile.babel.js，这是因为这个文件中用了`ES6`的语法，gulp会自动调用babel插件转成`ES5`语法。在这个文件中，定义我们的各种需求任务。

    import gulp from 'gulp';
    gulp.task('default',function() => {
        console.log('hello world');
    });

上面我们定义的一个最简单的任务, 名字为default, 就是打印一个hello;  这个任务没有用到其他插件，还没体现流的概念。

要运行gulp任务，只需切换到存放gulpfile.js文件的目录，然后在命令行中执行gulp命令就行了，gulp后面可以加上要执行的任务名，例如gulp task1，如果没有指定任务名，则会执行任务名为`default`的默认任务。

#### gulp API
要掌握gulp 的使用一点不难，我认为只要记住下面的几个API差不多了：

* gulp.task(name[, deps], fn)用来定义任务
  * name：任务名
  * deps：是当前定义的任务需要依赖的其他任务，为一个数组。当前定义的任务会在所有依赖的任务执行完毕后才开始执行。如果没有依赖，则可省略这个参数；
  * fn 为任务函数，我们把任务要执行的代码都写在里面。该参数也是可选的。

* gulp.src(globs[, options])定义所要读取的原文件路径，然后返回一个可以传递给插件的数据流。
  * globs：文件匹配模式，string或者array类型
  * options为可选参数；

举例：

        gulp.src("js/a.js");              //指定js目录下a.js文件
        gulp.src("js/*.js");              //指定js目录下任意js文件
        gulp.src("js/**/*.js");           //指定js目录下所有js文件,包括任意子目录
        gulp.src(["js/**/*.js","!js/**/*.min.js"])   //指定js目录下所有非压缩js文件


* .pipe()管道命令，用来链接gulp插件的，可以理解为将操作加入执行队列。所谓管道，就是将上一个命令的输出重定向到下一个命令的输入，一个插件处理完传到下一个插件

* gulp.dest(path[,options])定义用来指明处理完以后的文件存放路径，它将前面管道的输出写入文件，同时将这些输出继续输出下一级。
   * path 写入文件的路径；
   * options 可选参数，如指定options.mode（文件的权限值）

* gulp.watch(glob[, opts], tasks)监视文件的变化，并且运行相应的tasks
   * glob 为要监视的文件匹配模式，规则和用法与gulp.src()方法中的glob相同；
   * opts 为一个可选的配置对象，通常不需要用到。
   * tasks 为文件变化后要执行的任务，为一个数组。
* gulp.watch(glob[, opts, cb])，watch另外一种方式，传人回调函数function（event）作为参数
* gulp.run(tasks)使用代码执行任务。任务是并发执行的。

我们摒弃yo生成的gulpfile.babel.js帮我们生成的任务，而创建几个入门task，在这之前得先介绍几个任务中需要的插件：

* gulp-load-plugins自动帮你加载package.json文件里的gulp插件
* main-bower-files 读取bower安装的文件
* gulp-minify-css 压缩css
* gulp-minify-html 压缩html
* gulp-uglify 压缩js
* gulp-jshint(需要同时安装jshint) js语法静态检查
* gulp-concat 合并多个文件
* gulp-imagemin 压缩图片
* gulp-rename 重命名文件,通常压缩后的带.min后缀
* gulp-flatern 改变文件的路径层级
* del 清理文件或目录

可以通过`npm install --save-dev [插件名称]`来安装。

### 构建任务
我们对项目中的js,css,html文件进行压缩，开启一个静态server服务，监听源文件的变化，安装项目。

1. 加载插件
引入插件有3步，安装-加载-使用，加载方式如下：
在ES6语法里面可以使用

        import gulp from 'gulp';
        import gulpLoadPlugins from 'gulp-load-plugins';

    而在ES5中的方式是这样的

        var gulp=require('gulp');
        var gulpLoadPlugins=require('gulp-load-plugins');

    一个个插件很麻烦，于是就有gulp-load-plugins插件（上面引入的），可以加载package.json文件中所有的gulp模块。gulp-load-plugins加载后执行：

        var $ = gulpLoadPlugins({pattern: "*"})；
就加载了所有的插件。`$`可以用其它变量名表示。然后我们要使用gulp-rename和gulp-minify-html这两个插件的时候，就可以使用$.rename和$.minifyHtml来代替了,也就是原始插件名去掉gulp-前缀，之后再转换为驼峰命名。但是要注意 **pattern默认值是['gulp-*', 'gulp.*']**，如果gulpLoadPlugins不修改pattern参数，只加载名字是gulp开头的插件。

2. 压缩css

        gulp.task("styles",function() {
            return gulp.src("app/styles/*.css")
            .pipe($.minifyCss())
            .pipe($.rename({suffix:".min"}))
            .pipe(gulp.dest("dist/styles"));
        });
上面的任务流程一目了然，gulp.src读入源文件的所有css文件变成一个对象流，用pipe管道传入到gulp-minify-css插件，由它进行css压缩，完成后传入下一级的插件gulp-rename将文件名添加一个min的后缀，最后把文件保存到dist/styles目录下。

3. js语法检查、合并、压缩

        gulp.task("scripts",function() {
            return gulp.src("app/scripts/**/*.js") //读取js文件
            .pipe($.jshint())  //js语法检查
            .pipe($.jshint.reporter('default'))  //检查结果输出
            .pipe($.concat('main.js'))   //合并所有js文件到main.js
            .pipe($.uglify())   // 压缩js
            .pipe($.rename({suffix:".min"}))  //重命名
            .pipe(gulp.dest("dist/scripts"));  //输出
        });
这个任务把app源码目录下的所有js文件合并成一个main.js，然后对main.js进行压缩输出。

4. 读取bower_components
通过bower安装的第三方库，除了js,css文件外，往往有很多其他目录和文件，如何读取主要的css,jss到安装目录呢，我们借助glup-flatern, main-bower-files这两个插件。

    `main-bower-files`这个插件会读取bower.json的`dependencies`, 读取每个package的`main`属性，根据这些属性读取对应的文件。

    `gulp-flatern`这个插件可以把文件的路径根据需要变短，如a/b/c/jquery.js, 可以变为jquery.js, b/jquery.js, c/jquery.js, b/c/jquery.js。

        gulp.task('bowerfiles', function() {
            return gulp.src($.mainBowerFiles(),{ base: './bower_components' })
                .pipe($.flatten({ includeParents: 1})) //保留一层路径
                .pipe(gulp.dest("dist/bower_components"))
    });
执行后得到的路径类似如下：

        dist── bower_components
                ├── bootstrap
                │   ├── bootstrap.js
                │   ├── bootstrap.min.js
                │   ├── bootstrap.css
                │   └── bootstrap.min.css
                └── jquery
                    └── jquery.js


5. 图片压缩
图片压缩使用了gulp-imagemin插件和gulp-cache插件，鉴于压缩图片时比较耗时，而且不常修改，我们使用”gulp-cache插件加快构建过程，只压缩修改的图片，没有修改的图片直接从缓存文件读取。

        gulp.task('images', function() {
          return gulp.src('app/images/**/*')
            .pipe($.cache($.imagemin({
              progressive: true,
              interlaced: true,
              // don't remove IDs from SVGs, they are often used
              // as hooks for embedding and styling
              svgoPlugins: [{cleanupIDs: false}]
            })))
            .pipe(gulp.dest('dist/images'));
        });

6. 清理任务
构建一个任务，用来清理临时文件，安装目录，使用了gulp-del插件：

        var del = require('del');
        gulp.task('clean', function() {
          del('dist');
        });

7. 默认任务
默认任务可以简单的执行已存在的任务

        gulp.task("default", ["styles","scripts","bowerfiles","images"],function() {
            console.log("gulp task finished!");
        });

8. 监控任务
监控任务一般配合一个静态服务执行，也就是说要一个常驻进程才能有效果。执行gulp watch任务，当配置的目录发生变化时，执行相应的任务。

        gulp.task("watch",function() {
            gulp.watch("app/**/*.css",["styles"]);
            gulp.watch("app/**/*.js",["scripts"]);
            gulp.watch("app/images/*.",["images"]);
            gulp.watch("bowerfiles/**/.*", ["bowerfiles"]);
        });


gulpfile.js编辑完成后，只要输入`gulp 『任务名』`就可以执行任务了，如果执行过程中遇到莫名其妙的错误，可能是node-modules有些插件没安装成功，把node-modules目录删除，执行`npm install`重新安装，然后执行任务即可。

上面介绍的都是gulp入门级的任务，gulp的功能远不止这些，当然需要你了解更多插件的功能，甚至还可以自己开发所需的插件，组合运用起来完成更复杂的需求。

###和后端程序的结合实践
总的来说，一个web项目简单分为前端静态资源和后端服务程序，下面是我常用的project目录，结构清晰的表明这两部分。app后端业务程序将数据填入templates的页面中，返回给客户端, 完成动态交互部分；app这部分一般是打包然后安装到运行环境中，以模块的方式调用执行，static这部分其实就是yo产生的webapp部分，由gulp进行构建。整个project可以写个Makefile文件集合打包、构建、部署的命令，方便我们一键部署。

    project
    |-- app
    |   |-- __init__.py
    |   |-- templates
    |   |   |-- home.html
    |   |   |-- user.html
    |   |   `-- ....
    |   |-- model
    |   |   |-- __init__.py
    |   |   `-- ....
    |   |-- views
    |   |   |-- __init__.py
    |   |   `-- ....
    |   |-- urls.py
    |   `-- wsgi.py
    |
    |
    |-- static
    |   |-- src
    |   |     |-- robots.txt
    |   |     |-- index.html
    |   |     |-- styles
    |   |     |    |
    |   |     |    `-- *.css
    |   |     `-- scripts
    |   |         |
    |   |         `-- *.js
    |   |
    |   |-- bower.json
    |   |-- bower_components
    |   |     |-- bootstrap
    |   |     |-- jquery
    |   |     `-- ....
    |   |
    |   |-- gulpfile.babel.js
    |   |-- package.json
    |   |-- node_modules
    |   |-- gulp
    |   |-- gulpuglify
    |   `-- ....
    |
    |-- setup.py
    |-- Makefile
    `-- ...

部署后运行环境目录变成：

    project
    |-- var
    |   |-- www
    |   |   |-- index.html
    |   |   |-- js
    |   |   |-- css
    |   |   |-- images
    |   |   `-- ...
    |   |-- logs
    |   `-- ...
    |-- etc
    `-- ...

在web服务器中，对于访问静态资源的请求配置直接指向项目var/www的路径。要注意的后端程序中templates中使用到静态资源的路径必须为web服务器中的虚拟路径，这样才能正确找到对应的文件。

### 总结
本文只阐述了Yeoman自动化构建的内容，其他与异步加载的结合，ES6规范使用等并未过多叙述。前端的技术发展越来越成熟了，汲取一点已够我受用，还有更强大的功能等待我发掘和引用，进一步整合工程的最佳实践，优化web性能。前端发展太快了，至于本文先所阐述的前端构建和优化技术，或许已经有了更好的替代方案，anyway，欢迎大家来指正。

### 参考：

http://yeoman.io/authoring/index.html
https://www.smashingmagazine.com/2014/06/building-with-gulp/
http://www.w3ctrain.com/2015/12/22/gulp-for-beginners/
http://www.eguneys.com/blog/2014/09/17/lets-build-a-yeoman-generator-project-template-slash-w-gulp
http://gulpjs.com/plugins/