事情是这样的，再写一个http客户端帮助类的时候，由于设置属性的时候，我使用了个 $attr 数组，并使用__set()与__get()魔术方法进行设置和获取，但是我在其他类中调用的时候，判断某个变量是否为空是使用了
```
if(empty($client->url)) $client->url = 'xxxxxxx';
```
结果每次都会去执行这句话，就这个现象，进行讨论。
以下是四种不同的情况
```
class HttpCli
{
    protected $attr = [];

    public function __set($name, $value)
    {
        $this->attr[$name] = $value;
    }

    public function __get($name)
    {
        return $this->attr[$name];
    }

}

$http_cli = new HttpCli();
$http_cli->url = 'https://www.baidu.com';
$http_cli->method = 'GET';

var_dump(isset($http_cli->url));
var_dump(empty($http_cli->url));
var_dump($http_cli->url);
var_dump($http_cli);
# 结果
bool(false)
bool(true)
string(21) "https://www.baidu.com"
object(app\index\controller\HttpCli)#34 (1) {
  ["attr":protected]=>
  array(2) {
    ["url"]=>
    string(21) "https://www.baidu.com"
    ["method"]=>
    string(3) "GET"
  }
}
```
```
class HttpCli1
{
    protected $attr = [];

    public function __set($name, $value)
    {
        $this->attr[$name] = $value;
    }

    public function __get($name)
    {
        return $this->attr[$name];
    }

    /**
     * 解决方案
     */
    public function __isset($name)
    {
        if(!isset($this->attr[$name])) return null;
        return false === empty($this->attr[$name]);
    }
}

$http_cli = new HttpCli1();
$http_cli->url = 'https://www.baidu.com';
$http_cli->method = 'GET';

var_dump(isset($http_cli->url));
var_dump(empty($http_cli->url));
var_dump($http_cli->url);
var_dump($http_cli);

bool(true)
bool(false)
string(21) "https://www.baidu.com"
object(app\index\controller\HttpCli1)#35 (1) {
  ["attr":protected]=>
  array(2) {
    ["url"]=>
    string(21) "https://www.baidu.com"
    ["method"]=>
    string(3) "GET"
  }
}
```
```
class HttpCli2
{
    protected $url = '';
    protected $method = '';

    public function __set($name, $value)
    {
        $this->$name = $value;
    }

    public function __get($name)
    {
        return $this->$name;
    }
}

$http_cli = new HttpCli2();
$http_cli->url = 'https://www.baidu.com';
$http_cli->method = 'GET';

var_dump(isset($http_cli->url));
var_dump(empty($http_cli->url));
var_dump($http_cli->url);
var_dump($http_cli);

bool(false)
bool(true)
string(21) "https://www.baidu.com"
object(app\index\controller\HttpCli2)#34 (2) {
  ["url":protected]=>
  string(21) "https://www.baidu.com"
  ["method":protected]=>
  string(3) "GET"
}
```
```
class HttpCli3
{
    public $url = '';
    public $method = '';
}

$http_cli = new HttpCli3();
$http_cli->url = 'https://www.baidu.com';
$http_cli->method = 'GET';

var_dump(isset($http_cli->url));
var_dump(empty($http_cli->url));
var_dump($http_cli->url);
var_dump($http_cli);

bool(true)
bool(false)
string(21) "https://www.baidu.com"
object(app\index\controller\HttpCli3)#35 (2) {
  ["url"]=>
  string(21) "https://www.baidu.com"
  ["method"]=>
  string(3) "GET"
}
```

从php语言的角度而言，在操作一个不可访问属性的时候，会使用到PHP的重载方法（属性重载只能在对象中进行）
1.在给不可访问属性赋值时，__set() 会被调用。
2.读取不可访问属性的值时，__get()会被调用。
3.当对不可访问属性调用 isset()或 empty()时，__isset()会被调用。
4.当对不可访问属性调用 unset()时，__unset()会被调用。

实际上相当于调用了不可见属性的时候使用，获取值是没问题的，只是当对属性isset()或 empty()时，会调用__isset()，那既然知道是这样的话，我们只需要像例2:类HttpCli1那样的去实现下__isset()即可
```
public function __isset($name)
{
    if(!isset($this->attr[$name])) return null;
    return false === empty($this->attr[$name]);
}
```