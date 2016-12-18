title: Building sqlite3 with sqlcipher for node-webkit on Windows 10
date: 2016-12-17 23:46:54
tags: [sqlcipher node-webkit sqlite3 加密]
---

做的node-webkit项目好几年都是用的sqlite裸奔数据库，之前一直没有人能去解决数据库加密问题。
最近心情不错，就研究了下node-webkit平台下如何使用加密sqlite3。   
目前市面上可选择的免费加密方式并不多，其中SQLCipher提供的SQLite数据库的透明加密功能。   
然而SQLCipher虽然开源，但是目前市面上可找到的信息寥寥无几，因此官方就对编译好的SQLCipher进行收费，所以还是自己动手丰衣足食。

<!-- more -->

## 准备

* node.js - node.js 7.2.1 npm 3.10.10 64bit
* visual studio - 2015
* Activestate Perl - 5.24.0 32bit
* nasm - 2.12.02 32bit
* [MinGW](https://sourceforge.net/projects/mingw/files/Installer/) - 0.6.2-beta   install mingw32-base msys-base gcc-g++ tclsh
* nw-gyp `npm i nw-gyp -g` - 3.4.0
* node-pre-gyp `npm i node-pre-gyp -g` - 0.6.32
* node-webkit - 0.11.0 32bit

## node-sqlite3

安装sqlite3 `npm i sqlite3`， 进入 **'$(PROJECT)\node_modules\sqlite3\deps'** 解压gz文件到当前目录生成文件夹(sqlite-autoconf-3150000)

## OpenSSL

### options 1

下载 [OpenSSL source](https://github.com/openssl/openssl/tree/OpenSSL_1_0_2-stable) - 1.0.2 (注意: 1.1.0之后的版本不再带ssleay32.lib和libeay32.lib，请选择1.0.2 下载)
进入visual studio 2015 开发人员命令提示 （位于开始菜单菜单）
进到到OpenSSL 源码目录
运行命令
```bash
perl Configure VC-WIN32 --prefix=C:\OpenSSL-Win32
ms\do_nasm
nmake -f ms\nt.mak
nmake -f ms\nt.mak install
nmake -f ms\ntdll.mak
nmake -f ms\ntdll.mak install
```

### options 2

我已经编译好的 [OpenSSl](http://7lrzuu.com1.z0.glb.clouddn.com/OpenSSL-Win32.7z) - 1.0.2

### options 3

下载其他版本[Building OpenSSL with Visual Studio](http://p-nand-q.com/programming/windows/building_openssl_with_visual_studio_2013.html)  

1. 安装完成后复制头文件 **'C:\OpenSSL-Win32\include\openssl'** 到 **'$(PROJECT)\node_modules\sqlite3\deps\sqlite-autoconf-3150000\openssl'**
2. 复制 **'C:\OpenSSL-Win32\lib'** 文件夹内的文件 (需有`libeay32.lib` 和 `ssleay32.lib`) 到 **'$(PROJECT)\node_modules\sqlite3\deps\sqlite-autoconf-3150000\OpenSSL-Win32'**

## sqlcipher

### options 1

1. 下载[sqlcipher srouce](https://github.com/sqlcipher/sqlcipher) - 3.4.0 并解压
2. 将编译好OpenSSL目录内的 **'lib'** 和 **'bin'** 目录内的内容（ssleay32.dll libeay32.dll ssleay32.lib libeay32.lib 等） 复制到 sqlcipher 目录 
3. 运行 **'C:\MinGW\msys\1.0\msys.bat进入sqlcipher'** 目录 (cd c:\sqlcipher-master)
4. 运行
```bash
./configure --with-crypto-lib=none --disable-tcl CFLAGS="-DSQLITE_HAS_CODEC -DSQLCIPHER_CRYPTO_OPENSSL -I/c/OpenSSL-Win32/include /c/sqlcipher-master/libeay32.dll -L/c/sqlcipher-master/ -static-libgcc" LDFLAGS="-leay32"
make clean
make sqlite3.c
make
make dll
```

5. 测试
```bash
    sqlcipher test.db
    sqlcipher>PRAGMA key = 'password';
    sqlcipher>create table testtable (id int, name varchar(20));
    sqlcipher>insert into testtable (id,name) values (0,'alice'), (1,'bob'), (2,'charlie');
    sqlcipher>select * from testtable
        0:alice
        1:bob
        2:charlie
    sqlcipher>.exit
```

### options 2
下载已经编译好的 [sqlcipher windows lib](http://7lrzuu.com1.z0.glb.clouddn.com/sqlcipher-master.7z) - 3.4.0 

## 生成gz文件

1. 复制sqlcipher目录下的`sqlite3.h`  `sqlite3.c` `sqlite3ext.h` 到 **'$(PROJECT)\node_modules\sqlite3\deps\sqlite-autoconf-3150000'** 目录
2. 压缩 **'$(PROJECT)\node_modules\sqlite3\deps\sqlite-autoconf-3150000'** 文件夹 生成 **`sqlite-autoconf-3150000.tar.gz`** 文件

## 修改sqlite3.gyp

1. 修改 **'$(PROJECT)\node_modules\deps\sqlite3.gyp'** 文件，找到`'OS == "win"'`并在后添加
```
      ['OS == "win"', {
        'defines': [
          'WIN32'
        ],
        'link_settings': {
          'libraries': [
            '-llibeay32.lib',
            '-lssleay32.lib',
          ],
          'lirary_dirs': [
            '<(SHARED_INTERMEDIATE_DIR)/sqlite-autoconf-<@(sqlite_version)/OpenSSL-Win32'
          ]
        }
      }]
```
2. 在'target_name': 'sqlite3' 中找到'defines' 并添加
```
        'SQLITE_HAS_CODEC=1',
        'CODEC_TYPE=CODEC_TYPE_AES128',
        'SQLITE_CORE',
        'THREADSAFE',
        'SQLITE_SECURE_DELETE',
        'SQLITE_SOUNDEX',
        'SQLITE_ENABLE_COLUMN_METADATA'
```
添加后
```
      'defines': [
        '_REENTRANT=1',
        'SQLITE_THREADSAFE=1',
        'SQLITE_ENABLE_FTS3',
        'SQLITE_ENABLE_JSON1',
        'SQLITE_ENABLE_RTREE',

        'SQLITE_HAS_CODEC=1',
        'CODEC_TYPE=CODEC_TYPE_AES128',
        'SQLITE_CORE',
        'THREADSAFE',
        'SQLITE_SECURE_DELETE',
        'SQLITE_SOUNDEX',
        'SQLITE_ENABLE_COLUMN_METADATA'
      ],
```

##  编译

### options 1
回到 **'$(PROJECT)\node_modules\sqlite3'** 执行 `npm install --build-from-source --runtime=node-webkit --target_arch=ia32 --target=0.11.0`

### options 2
下载编译好的 [sqlite3-sqlcipher](http://7lrzuu.com1.z0.glb.clouddn.com/sqlite3-sqlcipher.7z)

## node-webkit测试

在node-webkit目录中新建 `main.html` 文件并加入写入以下javascript代码
```js
var sqlite3 = require('sqlite3').verbose();
var db = new sqlite3.Database('test.db');

db.serialize(function() {
  db.run("PRAGMA KEY = 'secret'");
  db.run("PRAGMA CIPHER = 'aes-128-cbc'");

  db.run("DROP TABLE IF EXISTS lorem");
  db.run("CREATE TABLE lorem (info TEXT)");

  var stmt = db.prepare("INSERT INTO lorem VALUES (?)");
  for (var i = 0; i < 10; i++) {
    stmt.run("Ipsum " + i);
  }
  stmt.finalize();

  db.each("SELECT rowid AS id, info FROM lorem", function(err, row) {
    document.write(row.id + " - " + row.info + '<br>');
  });
});

db.close();
```

修改 `package.json` 入口 运行`nw.exe`查看结果

[nw-0.11.0-ia32 with sqlcipher](http://7lrzuu.com1.z0.glb.clouddn.com/nw-0.11.0-ia32%20with%20sqlcipher.7z)

## 数据库加密与迁移

````js
db.serialize(function() {
    db.run("ATTACH DATABASE 'encrypted.db' AS encrypted KEY 'testkey'")
        
    // 可在此添加参数 如:
    db.run("PRAGMA encrypted.cipher = 'aes-256-cfb'")
    db.run("PRAGMA encrypted.cipher_page_size = 4096")
    // end
    
    db.run("SELECT sqlcipher_export('encrypted')")    
    db.run("DETACH DATABASE encrypted")
})
```


详见[sqlcipher api](https://www.zetetic.net/sqlcipher/sqlcipher-api/#cipher)

是不是感觉赚到了...
