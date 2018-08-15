title: DVWA练习4-上传漏洞
date: 2016-03-07 17:31:32
categories: 安全
tags: Web安全
---
### 一、上传漏洞
#### 1.简介
Web服务中经常有上传文件的功能，如果不对上传的文件进行限制或者限制被绕过，就有可能上传可执行文件，获取系统权限。
#### 2.漏洞成因
参考wooyun wiki  
*
导致文件上传的漏洞的原因较多，主要包括以下几类：  
服务器配置不当  
开源编辑器上传漏洞  
本地文件上传限制被绕过  
过滤不严或被绕过  
文件解析漏洞导致文件执行  
文件路径截断*

我们常见的有解析漏洞，00阶段，抓包改名等。
<!--more-->
#### 3.防御方法
在服务器后端对上传的文件进行过滤  
判断文件类型  
对文件重命名  
上传目录设置为不可执行  
使用随机数改写文件名和路径  

### 二、DVWA
#### 1.low
没有上传限制，直接可以上传木马。
源码分析：
```php
<?php
if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    //上传目录即$target_path为../../hackable/uploads/，其中DVWA_WEB_PAGE_TO_ROOT被定义为../../
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );
    //上传目录为../../hackable/uploads/+文件名
    // Can we move the file to the upload folder?
    if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
        // No 是否上传成功
        echo '<pre>Your image was not uploaded.</pre>';
    }
    else {
        // Yes!
        echo "<pre>{$target_path} succesfully uploaded!</pre>";
    }
}
?>
```
#### 2.Medium
上传必须为图片类型。方法一，首先选择一张图片，图片名10-170609_700.jpg点击上传，如图。
![](http://7xo8y2.com1.z0.glb.clouddn.com/uploadtupian.png)
使用burpsuite抓包。接下来可以使用00截断进行上传，在截取的数据中修改图片名称为"filename="10-170609_700.php.jpg"，点击hex,将[10-170609_700.php.jpg]中10-170609_700.php后的[.]对应的2e修改为00。如图。
![](http://7xo8y2.com1.z0.glb.clouddn.com/00jieduan.png)
修改完成后点击go即可上传成功。
![](http://7xo8y2.com1.z0.glb.clouddn.com/xiugaiwancheng.png)
如图，最终访问地址为../../hackable/uploads/10-170609_700.php
方法二，选择php文件上传，使用burpsuite拦截，修改文件类型为图片即可。
即将Content-Type: application/octet-stream修改为Content-Type: image/jpeg
![](http://7xo8y2.com1.z0.glb.clouddn.com/wenjianleixing.png)
![](http://7xo8y2.com1.z0.glb.clouddn.com/xiugaoiwenjianleixing.png)
源码分析：
```php
<?php
if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );
    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_type = $_FILES[ 'uploaded' ][ 'type' ];
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    // Is it an image?判断是否为jpeg或png图片，并且大小小于100000字节。
    if( ( $uploaded_type == "image/jpeg" || $uploaded_type == "image/png" ) &&
        ( $uploaded_size < 100000 ) ) {
        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}
?>
```
#### 3.High
这次使用00截断不能再上传成功。
源码分析：
```php
<?php
if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );
    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
    //strrpos() 函数查找字符串在另一字符串中最后一次出现的位置，substr() 函数返回字符串的一部分。
    //在这里得到文件名中[.]之后的字符串，也就是文件后缀
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    $uploaded_tmp  = $_FILES[ 'uploaded' ][ 'tmp_name' ];
    // Is it an image?判断文件后缀否为ipg,jpeg或png。并判断大小，
    if( ( strtolower( $uploaded_ext ) == "jpg" || strtolower( $uploaded_ext ) == "jpeg" || strtolower( $uploaded_ext ) == "png" ) &&
        ( $uploaded_size < 100000 ) &&
        getimagesize( $uploaded_tmp ) ) {
        //getimagesize如果指定的文件如果不是有效的图像，会返回 false
        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $uploaded_tmp, $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}
?>
```
上网查了查，有几个参考。
http://lcx.cc/?i=2158  
http://www.teafly.org/hack/228.html  
http://www.myhack58.com/Article/html/3/62/2015/60728.htm  
https://segmentfault.com/a/1190000003911296  
看到strrpos() 函数查找字符串在另一字符串中**最后一次**出现的位置，所以如果上传的是fliename.php.jpg，则$uploaded_ext=jpg,就可以通过上传，但是上传后还是图片类型。
另外 move_uploaded_file，getimagesize函数也有漏洞，但是在实际测试的时候和网上找的不符。
修改文件为a.php\0jpg，上传后就是 hackable/uploads/0jpg succesfully uploaded!。

更新：之前一直认为就算上传了图片木马也无法被执行。没有考虑到文件包含。这里可以上传图片木马，之后利用文件包含漏洞执行。例如：`http://127.0.0.1/dvwa/vulnerabilities/fi/?page=file:///E:/xampp/htdocs/dvwa/hackable/uploads/test.jpg`

#### 4.Impossible
查看源码：
```php
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Check Anti-CSRF token 防御CSRF
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );


    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    $uploaded_type = $_FILES[ 'uploaded' ][ 'type' ];
    $uploaded_tmp  = $_FILES[ 'uploaded' ][ 'tmp_name' ];

    // Where are we going to be writing to?
    $target_path   = DVWA_WEB_PAGE_TO_ROOT . 'hackable/uploads/';
    //$target_file   = basename( $uploaded_name, '.' . $uploaded_ext ) . '-';
    $target_file   =  md5( uniqid() . $uploaded_name ) . '.' . $uploaded_ext;
    //uniqid() 函数基于以微秒计的当前时间，生成一个唯一的 ID。
    //md5() 函数计算字符串的 MD5 散列。对上传文件进行重命名。
    $temp_file     = ( ( ini_get( 'upload_tmp_dir' ) == '' ) ? ( sys_get_temp_dir() ) : ( ini_get( 'upload_tmp_dir' ) ) );
    $temp_file    .= DIRECTORY_SEPARATOR . md5( uniqid() . $uploaded_name ) . '.' . $uploaded_ext;

    // Is it an image? 是否为图片
    if( ( strtolower( $uploaded_ext ) == 'jpg' || strtolower( $uploaded_ext ) == 'jpeg' || strtolower( $uploaded_ext ) == 'png' ) &&
        ( $uploaded_size < 100000 ) &&
        ( $uploaded_type == 'image/jpeg' || $uploaded_type == 'image/png' ) &&
        getimagesize( $uploaded_tmp ) ) {

        // Strip any metadata, by re-encoding image (Note, using php-Imagick is recommended over php-GD) 创建新图像
        if( $uploaded_type == 'image/jpeg' ) {
            $img = imagecreatefromjpeg( $uploaded_tmp );
            imagejpeg( $img, $temp_file, 100);
        }
        else {
            $img = imagecreatefrompng( $uploaded_tmp );
            imagepng( $img, $temp_file, 9);
        }
        imagedestroy( $img );

        // Can we move the file to the web root from the temp folder?
        if( rename( $temp_file, ( getcwd() . DIRECTORY_SEPARATOR . $target_path . $target_file ) ) ) {
            // Yes!
            echo "<pre><a href='${target_path}${target_file}'>${target_file}</a> succesfully uploaded!</pre>";
        }
        else {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }

        // Delete any temp files
        if( file_exists( $temp_file ) )
            unlink( $temp_file );
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```
