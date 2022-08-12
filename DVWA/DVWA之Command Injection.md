**Command Injection**

Command Injection，即命令注入，是指通过提交恶意构造的参数破坏命令语句结构，从而达到执行恶意命令的目的。PHP命令注入攻击漏洞是PHP应用程序中常见的脚本漏洞之一，国内著名的Web应用程序Discuz!、DedeCMS等都曾经存在过该类型漏洞。

[![漏洞提醒.png](http://image.3001.net/images/20161016/14765776616648.png!small)](http://image.3001.net/images/20161016/14765776616648.png)

**下面对四种级别的代码进行分析。**

**Low**

服务器端核心代码

```
<?php        
if( isset( $_POST[ 'Submit' ]  ) ) {        
    // Get input        
    $target = $_REQUEST[ 'ip' ];        
    // Determine OS and execute the ping command.        1
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {        1
        // Windows        1
        $cmd = shell_exec( 'ping  ' . $target );        1
    }        1
    else {        2
        // *nix        2
        $cmd = shell_exec( 'ping  -c 4 ' . $target );        2
    }        2
    // Feedback for the end user        2
    echo "<pre>{$cmd}</pre>";        3
}        3
?>
```


相关函数介绍 

_stristr(string,search,before\_search)_

stristr函数搜索字符串在另一字符串中的第一次出现，返回字符串的剩余部分（从匹配点），如果未找到所搜索的字符串，则返回FALSE。参数string规定被搜索的字符串，参数search规定要搜索的字符串（如果该参数是数字，则搜索匹配该数字对应的ASCII值的字符），可选参数before\_true为布尔型，默认为“false”，如果设置为“true”，函数将返回search参数第一次出现之前的字符串部分。

_php\_uname(mode)_

这个函数会返回运行php的操作系统的相关描述，参数mode可取值”a” （此为默认，包含序列”s n r v m”里的所有模式），”s”（返回操作系统名称），”n”（返回主机名），” r”（返回版本名称），”v”（返回版本信息）， ”m”（返回机器类型）。

可以看到，服务器通过判断操作系统执行不同ping命令，但是对ip参数并未做任何的过滤，导致了严重的命令注入漏洞。

**漏洞利用**

window和linux系统都可以用&&来执行多条命令

127.0.0.1&&net user

[![漏洞利用](http://image.3001.net/images/20161016/14765776761898.png!small)](http://image.3001.net/images/20161016/14765776761898.png)

Linux下输入127.0.0.1&&cat /etc/shadow甚至可以读取shadow文件，可见危害之大。

**Medium**

服务器端核心代码

```
<?php        
if( isset( $_POST[ 'Submit' ]  ) ) {        
    // Get input        
    $target = $_REQUEST[ 'ip' ];        
    // Set blacklist        1
    $substitutions = array(        1
        '&&' => '',        1
        ';'  => '',        1
    );        1
    // Remove any of the charactars in the array (blacklist).        2
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );        2
    // Determine OS and execute the ping command.        2
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {        2
        // Windows        2
        $cmd = shell_exec( 'ping  ' . $target );        3
    }        3
    else {        3
        // *nix        3
        $cmd = shell_exec( 'ping  -c 4 ' . $target );        3
    }        4
    // Feedback for the end user        4
    echo "<pre>{$cmd}</pre>";        4
}        4
?>
```


可以看到，相比Low级别的代码，服务器端对ip参数做了一定过滤，即把”&&” 、”;”删除，本质上采用的是黑名单机制，因此依旧存在安全问题。

**漏洞利用**

1、127.0.0.1&net user

因为被过滤的只有”&&”与” ;”，所以”&”不会受影响。

[![漏洞利用 2](http://image.3001.net/images/20161016/14765777056753.png!small)](http://image.3001.net/images/20161016/14765777056753.png)

这里需要注意的是”&&”与” &”的区别：

Command 1&&Command 2

先执行Command 1，执行成功后执行Command 2，否则不执行Command 2

[![Command 1&&Command 2](http://image.3001.net/images/20161016/1476577730171.png!small)](http://image.3001.net/images/20161016/1476577730171.png)

Command 1&Command 2

先执行Command 1，不管是否成功，都会执行Command 2

[![先执行Command 1，不管是否成功，都会执行Command 2](http://image.3001.net/images/20161016/1476577748635.png!small)](http://image.3001.net/images/20161016/1476577748635.png)

2、由于使用的是str\_replace把”&&” 、”;”替换为空字符，因此可以采用以下方式绕过：

127.0.0.1&;&ipconfig

 [![127.0.0.1&;&ipconfig 绕过](http://image.3001.net/images/20161016/14765777625148.png!small)](http://image.3001.net/images/20161016/14765777625148.png)

这是因为”127.0.0.1&;&ipconfig”中的” ;”会被替换为空字符，这样一来就变成了”127.0.0.1&& ipconfig” ，会成功执行。

**High**

服务器端核心代码

```
<?php        
if( isset( $_POST[ 'Submit' ]  ) ) {        
    // Get input        
    $target = trim($_REQUEST[ 'ip' ]);        
    // Set blacklist        1
    $substitutions = array(        1
        '&'  => '',        1
        ';'  => '',        1
        '|  ' => '',        1
        '-'  => '',        2
        '$'  => '',        2
        '('  => '',        2
        ')'  => '',        2
        '`'  => '',        2
        '||' => '',        3
    );        3
    // Remove any of the charactars in the array (blacklist).        3
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );        3
    // Determine OS and execute the ping command.        3
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {        4
        // Windows        4
        $cmd = shell_exec( 'ping  ' . $target );        4
    }        4
    else {        4
        // *nix        5
        $cmd = shell_exec( 'ping  -c 4 ' . $target );        5
    }        5
    // Feedback for the end user        5
    echo "<pre>{$cmd}</pre>";        5
}        6
?>
```


相比Medium级别的代码，High级别的代码进一步完善了黑名单，但由于黑名单机制的局限性，我们依然可以绕过。

**漏洞利用**

黑名单看似过滤了所有的非法字符，但仔细观察到是把”| ”（注意这里|后有一个空格）替换为空字符，于是 ”|”成了“漏网之鱼”。

127.0.0.1|net user

[![127.0.0.1|net user 利用](http://image.3001.net/images/20161016/14765777809564.png!small)](http://image.3001.net/images/20161016/14765777809564.png)

Command 1 | Command 2

“|”是管道符，表示将Command 1的输出作为Command 2的输入，并且只打印Command 2执行的结果。

**Impossible**

服务器端核心代码

```
<?php        
if( isset( $_POST[ 'Submit' ]  ) ) {        
    // Check Anti-CSRF token        
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );        
    // Get input        1
    $target = $_REQUEST[ 'ip' ];        1
    $target = stripslashes( $target );        1
    // Split the IP into 4 octects        1
    $octet = explode( ".", $target );        1
    // Check IF each octet is an integer        2
    if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) ) {        2
        // If all 4 octets are int's put the IP back together.        2
        $target = $octet[0] . '.' . $octet[1] . '.' . $octet[2] . '.' . $octet[3];        2
        // Determine OS and execute the ping command.        2
        if( stristr( php_uname( 's' ), 'Windows NT' ) ) {        3
            // Windows        3
            $cmd = shell_exec( 'ping  ' . $target );        3
        }        3
        else {        3
            // *nix        4
            $cmd = shell_exec( 'ping  -c 4 ' . $target );        4
        }        4
        // Feedback for the end user        4
        echo "<pre>{$cmd}</pre>";        4
    }        5
    else {        5
        // Ops. Let the user name theres a mistake        5
        echo '<pre>ERROR: You have entered an invalid IP.</pre>';        5
    }        5
}        6
// Generate Anti-CSRF token        6
generateSessionToken();        6
?>
```


相关函数介绍

_stripslashes(string)_

stripslashes函数会删除字符串string中的反斜杠，返回已剥离反斜杠的字符串。

_explode(separator,string,limit)_

把字符串打散为数组，返回字符串的数组。参数separator规定在哪里分割字符串，参数string是要分割的字符串，可选参数limit规定所返回的数组元素的数目。

_is\_numeric(string)_

检测string是否为数字或数字字符串，如果是返回TRUE，否则返回FALSE。

可以看到，Impossible级别的代码加入了Anti-CSRF token，同时对参数ip进行了严格的限制，只有诸如“数字.数字.数字.数字”的输入才会被接收执行，因此不存在命令注入漏洞。

**本文原创作者：lonehand**

**转自：[http://www.freebuf.com/articles/web/116714.html](http://www.freebuf.com/articles/web/116714.html)**